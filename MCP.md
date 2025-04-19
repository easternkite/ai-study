# MCP
## MCP란? 
MCP는 AI모델이 외부 데이터 소스나 도구와 통신할 수 있도록 표준화된 프로토콜이다.
여기서 표준화라는 것은 플러그인 처럼 떼었다 붙였다 하기 용이한 형태로 제공된다고 이해하면 된다.

### 어떻게 MCP는 동작할까?
MCP는 Client-Server 아키텍쳐처 따른다.
* Clients (LLM 클라이언트 앱)는 Server와 연결하여 특정 데이터를 요청한다.
* Server (MCP tool Providers) tool, API, 데이터소스를 client에게 제공한다.

### MCP의 이점?
1. 표준화된 통합 : LLM을 어떤 외부 시스템과 손쉽게 결합할 수 있다.
2. 유연성 : LLM들은 수요에 따라 다양한 툴과 서비스를 이용할 수 있다.
3. 보안성 : 별도의 하드코딩없이 안전한 API 상호작용을 지원한다.
4. 간단한 개발 : MCP 서버의 빌드 및 추출이 간편하다 
5. 쉬운 유지보수 : 반복적인 로직이 필요없다.

### 아키텍처
MCP 는 자연어 입력을 LLM(Gemini)을 통해 가공하고, MCP 서버를 통해 실제 데이터를 트리거할 수 있는 구조다.
![](https://firebasestorage.googleapis.com/v0/b/brocallie.appspot.com/o/pasterly%2Fimage_1745072097449_ctjdg2.png?alt=media&token=a30bc84a-6894-4289-8f8c-00d42813c440)

1. User to Client : 사용자는 자연어를 통해 쿼리한다.
2. Client to Gemini : Gemini에게 자연어 처리를 요청하여 실제 데이터를 사용할 수 있는 구조로 가공한다.
3. Gemini to Client : 가공된 데이터를 Client에게 리턴한다.
4. Client to MCP Server : MCP 서버에 데이터를 건내주며, 원하는 서비스 데이터를 요청한다.
5. MCP Server to External Tool : 외부 서비스를 호출한다.
6. External Tool to MCP Server : MCP Server에 실제 데이터를 전달한다.
7. MCP Server to Client : Client에 실제 데이터를 리턴한다.
8. Client to User : Client는 사용자에게 최종 데이터를 제공한다.

## 간단한 구현 (mcp-flight-search 이용)
```python
import os
import asyncio
import json

from google import genai
from google.genai import types
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

client = genai.Client(api_key=os.getenv("GEMINI_API_KEY"))

server_params = StdioServerParameters(
    command="mcp-flight-search",
    args=["--connection_type", "stdio"],
    env={"SERP_API_KEY": os.getenv("SERP_API_KEY")},
)

async def run():
    prompt = input("✈️ 검색할 항공편 정보를 자연어로 입력하세요 (예: 'Find flights from Seoul to Tokyo on 2025-05-05'): ")
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            mcp_tools = await session.list_tools()
            tools = [
                types.Tool(
                    function_declarations=[
                        {
                            "name": tool.name,
                            "description": tool.description,
                            "parameters": {
                                k: v
                                for k, v in tool.inputSchema.items()
                                if k not in ["additionalProperties", "$schema"]
                            },
                        }
                    ]
                )
                for tool in mcp_tools.tools
            ]
            response = client.models.generate_content(
                model="gemini-2.5-pro-exp-03-25",
                contents=prompt,
                config=types.GenerateContentConfig(
                    temperature=0,
                    tools=tools,
                ),
            )
            print(response)

            function_call = response.candidates[0].content.parts[0].function_call

            result = await session.call_tool(
                function_call.name, arguments=dict(function_call.args)
            )

            pretty_print_flights(result.content)

def pretty_print_flights(contents):
    for i, item in enumerate(contents, 1):
        try:
            data = json.loads(item.text)
            print(f"✈️ Flight {i}")
            print(f"  Airline      : {data['airline']}")
            print(f"  Price        : ${data['price']}")
            print(f"  Duration     : {data['duration']}")
            print(f"  Stops        : {data['stops']}")
            print(f"  Departure    : {data['departure']}")
            print(f"  Arrival      : {data['arrival']}")
            print(f"  Class        : {data['travel_class']}")
            print(f"  Logo URL     : {data['airline_logo']}")
            print("-" * 40)
        except json.JSONDecodeError:
            print(f"⚠️ Could not parse flight {i} content:", item.text)
            

asyncio.run(run())

```
