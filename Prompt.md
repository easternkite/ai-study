
더 효율적인 응답을 생성하기 위해 아래 컴포넌트를 추가하면 좋다.
* Persona + Objective : You are a **seasoned travel blogger** and guide with **a knack for unearthing hidden gems and creating unforgettable travel itineraries**.

* **Context** : **Customers are typically between 20-35 years old**, adventurous, budget-conscious and interested in outdoor activities. They are **looking for recommendations that are interesting and memorable**.

* **Instructions** : **Your task focuses on** trip inspiration, detailed planning, and seamless logistics based on the location the customer is interested in. **Document a potential user journey for finding, curating, and utilizing a travel itinerary designed for this specific location**.

* **Tone** : **Go beyond existing usual itineraries**, and suggest innovative ways to enhance the experience!

* **Format** : **Format this itinerary into a table** with columns Day, Location, Experiences, Things to know, and The How. The How column describes in detail how to accomplish the plan for the recommended experience.

* **Input** : Customer location: {user input specified here}

* **Prefilled response** : Itinerary:

## 프롬프트 작성 요령

### 1. 인스트럭션은 간결하게
> LLM은 간결한 지시사항을 잘 따릅니다.

```
_Prompt:_

**Assume you are a functional expert for text extraction. Extract the items from this transcript in JSON, and separate drinks from food.**

  

<Transcript>

Speaker 1 (Customer): Hi, can I get a burger and a large fry, please?

Speaker 2 (Employee): Coming right up! Anything else you'd like

to add to your order?

Speaker 1: Hmm, maybe a small lemonade. And could I get the fry with ketchup

on the side?

Speaker 2: No problem, one burger, one large fry with ketchup on the

side, and a small lemonade. That'll be $5.87. Drive through to the next window

please.

</Transcript>

  

_Response:_
{
    "food": [
        "burger", "large fry"
    ],
    "drinks": [
        "small lemonade"
    ]
}
```

### 명료하고 세분화된 인스트럭션을 구성하자

아래와 같은 프롬프트가 있다면 동작은 하지만 원하는대로 응답해주진 않는다.
```
Summarize the meeting notes.
```
요구하는 정보를 더 구체적으로 제시하면 좋다.
```
Summarize the meeting notes in a single paragraph. Then write a markdown list of the speakers and each of their key points. Finally, list the next steps or action items suggested by the speakers, if any.
```

### 페르소나를 적용해보자
모델에게 페르소나를 입히면 관련 질문에 대한 문맥에 더 집중하게 되면서 정확도가 올라간다.

다음과 같이 원하는 바를 디이렉트를 물어보는 것은 좋지 않다.
```
What is the most reliable Google Cloud load balancer?
```

그 대신에, 모델에게 페르소나를 적용하여, 해당 문맥에 특성화된 응답을 얻도록 유도할 수 있다.
```
You are a Google Cloud technical support engineer who specializes in cloud networking and responds to customer’s questions.

Question: What is the most reliable Google Cloud load balancer?
```

### Safety filter를 검증하자 
Safety filter를 적용하여 모델의 특정 위험한 문맥에 대한 응답을 제어할 수 있다.


### Temperature 를 변경해보자
Temperature 가 낮을 수록 교과서적인 응답, 높을 수록 창의력이 높은 응답을 유도할 수 있다.

### 예시는 들되, 최소화 하자
예시는 특정 형식으로 질문하도록 유도할 수 있지만, 많이 사용한다면 오히려 독이된다.
예시는 최소화하고 예시의 퀄리티를 높이는 방향으로 작성해야한다.

### 부정적인 인스트럭션이나 예시는 지양하자
예를들어 무엇을 하지말지 정하는 것 보다, 무엇을 할지에 대한 지시를 내리는게 더 퀄리티 높은 응답을 이끌 수 있다.

부정 예시
```
**The following is an agent that recommends movies to a customer.**

**DO NOT ASK FOR INTERESTS. DO NOT ASK FOR PERSONAL INFORMATION.**

**Customer: Please recommend a movie based on my interests.**

**Agent:**
```


긍정 예시
```
**The following is an agent that recommends movies to a customer.**

**The agent should recommend a movie from the top**

**global trending movies. It should refrain from asking users for their**

**preferences and avoid asking for personal information.**

**If the agent doesn't have a movie to recommend, it should**

**respond "Sorry, couldn't find a movie to recommend today.".**

**Customer: Please recommend a movie based on my interests.**

**Agent:**
```

### XML을 사용하여 더욱 명확하게 구분하자
구분자를 사용하는것은 의도하는 바를 의미론적으로 명확히 전달할 수 있다.
```
**You are a professional technical writer with excellent reading comprehending capabilities.**

**Your mission is to provide a coherent answer to the customer query by selecting unique sources from the document and organizing the response in a professional, objective tone. Provide your thought process to explain how you reasoned to provide the response.**

**Steps:**

**1. Read and understand the query and sources thoroughly.**

**2. Use all sources provided in the document to think about how to help the customer by providing a rational answer to their query.**

**3. If the sources in the document are overlapping or have duplicate details, select sources which are most detailed and comprehensive.**

**Follow the examples below:**

**<EXAMPLES>**

**<EXAMPLE>{example 1}</EXAMPLE>**

**<EXAMPLE>{example 2}</EXAMPLE>**

**</EXAMPLES>**

**Now it's your turn!**

**<DOCUMENT>  
**

**{context}**

**</DOCUMENT>**

  

**<INSTRUCTIONS>**

**Your response should include a 2-step cohesive answer with the following keys:**

**1. "Thought" key: Explain how you would use the sources in the document to partially or completely answer the query.**

**2. "Technical Document":**

    **- Prepend source citations in "{Source x}" format based on order of appearance.**

    **- Present each source accurately without adding new information.**

    **- Include at least one source in Technical Document; don't leave it blank.**

    **- Avoid mixing facts from different sources; use transitional phrases for flow.**

**3. The order of keys in the response must be "Thought" and "Technical Document."**

**4. Double-check compliance with all the instructions.**

**</INSTRUCTIONS>**

**<QUERY>{query}</QUERY>**

  

**OUTPUT:**
```

### 인스트럭션을 설계할 때 그 이유를 함께 응답하도록 유도하라
왜 그런 응답을 냈는지 이유를 함께 물어본다면 응답의 정확성은 올라간다.
