![image](https://github.com/kyopark2014/llama3-rag/assets/52392004/38932dde-7532-41a6-ac5e-4aa914ec5f2f)# llama3-rag

전체적인 Architecture는 아래와 같습니다.

<img src="./images/basic-architecture.png" width="800">
   
## System Design

### LangChain

```python
boto3_bedrock = boto3.client(
    service_name='bedrock-runtime',
    region_name=bedrock_region,
    config=Config(
        retries = {
            'max_attempts': 30
        }
    )
)
parameters = {
    "max_gen_len": 1024,  
    "top_p": 0.9, 
    "temperature": 0.1
}    

chat = ChatBedrock(   
    model_id=modelId,
    client=boto3_bedrock, 
    model_kwargs=parameters,
)
```

### Basic Chat

```python
def general_conversation(connectionId, requestId, chat, query):
    if isKorean(query)==True :
        system = (
            "다음의 Human과 Assistant의 친근한 이전 대화입니다. Assistant은 상황에 맞는 구체적인 세부 정보를 충분히 제공합니다. Assistant의 이름은 서연이고, 모르는 질문을 받으면 솔직히 모른다고 말합니다. 답변은 한국어로 합니다."
        )
    else: 
        system = (
            "Using the following conversation, answer friendly for the newest question. If you don't know the answer, just say that you don't know, don't try to make up an answer. You will be acting as a thoughtful advisor."
        )
    
    human = "{input}"
    
    prompt = ChatPromptTemplate.from_messages([("system", system), MessagesPlaceholder(variable_name="history"), ("human", human)])
    
    history = memory_chain.load_memory_variables({})["chat_history"]
                
    chain = prompt | chat    
    try: 
        isTyping(connectionId, requestId)  
        stream = chain.invoke(
            {
                "history": history,
                "input": query,
            }
        )
        msg = readStreamMsg(connectionId, requestId, stream.content)    
                            
        msg = stream.content
        print('msg: ', msg)
    except Exception:
        err_msg = traceback.format_exc()
        print('error message: ', err_msg)        
        
    return msg
```

여기서 Stream은 아래와 같이 event를 추출하여 json format으로 client에 결과를 전달합니다. 

```python
def readStreamMsg(connectionId, requestId, stream):
    msg = ""
    if stream:
        for event in stream:
            msg = msg + event

            result = {
                'request_id': requestId,
                'msg': msg,
                'status': 'proceeding'
            }
            sendMessage(connectionId, result)
    return msg
```

### 대화 이력의 관리

사용자가 접속하면, DynamoDB에서 대화 이력을 가져옵니다. 이것은 최초 접속 1회만 수행합니다. 

```python
def load_chat_history(userId, allowTime):
    dynamodb_client = boto3.client('dynamodb')

    response = dynamodb_client.query(
        TableName=callLogTableName,
        KeyConditionExpression='user_id = :userId AND request_time > :allowTime',
        ExpressionAttributeValues={
            ':userId': {'S': userId},
            ':allowTime': {'S': allowTime}
        }
    )


```

Context에 넣을 history를 가져와서 memory_chain에 등록합니다.

```pytho
for item in response['Items']:
    text = item['body']['S']
    msg = item['msg']['S']
    type = item['type']['S']

    if type == 'text' and text and msg:
        memory_chain.chat_memory.add_user_message(text)
        if len(msg) > MSG_LENGTH:
            memory_chain.chat_memory.add_ai_message(msg[:MSG_LENGTH])                          
        else:
            memory_chain.chat_memory.add_ai_message(msg) 
```

Lambda와 같은 서버리스는 이벤트가 있을 경우에만 사용이 가능하므로, 이벤트의 userId를 기준으로 메모리를 관리합니다. 

map_chain = dict()

```python
if userId in map_chain:  
    memory_chain = map_chain[userId]    
else: 
    memory_chain = ConversationBufferWindowMemory(memory_key="chat_history", output_key='answer’,
              return_messages=True, k=10)
    map_chain[userId] = memory_chain
```

새로운 입력(text)과 응답(msg)를 user/ai message로 저장합니다.

```python
memory_chain.chat_memory.add_user_message(text)
memory_chain.chat_memory.add_ai_message(msg)
```

### WebSocket Stream 사용하기 

#### Client

WebSocket을 연결하기 위하여 endpoint를 접속을 수행합니다. onmessage()로 메시지를 받습니다. WebSocket이 연결되면 onopen()로 초기화를 수행합니다. 일정 간격으로 keep alive 동작을 수행합니다. 네트워크 재접속 등의 이유로 세션이 끊어지면 onclose()로 확인할 수 있습니다.

```python
const ws = new WebSocket(endpoint);
ws.onmessage = function (event) {        
    response = JSON.parse(event.data)

    if(response.request_id) {
        addReceivedMessage(response.request_id, response.msg);
    }
};
ws.onopen = function () {
    isConnected = true;
    if(type == 'initial')
        setInterval(ping, 57000); 
};
ws.onclose = function () {
    isConnected = false;
    ws.close();
};
```



