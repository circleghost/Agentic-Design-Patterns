# 附錄 C - 智能體框架快速概覽

## LangChain

LangChain 是一個用於開發由大語言模型驅動的應用程式的框架。其核心優勢在於其 LangChain 表達語言 (LCEL)，它允許你將組件「管線化」連接成鏈。這建立了一個清晰的線性序列，其中一個步驟的輸出成為下一個步驟的輸入。它是為有向無環圖 (DAGs) 的工作流程而建造的，意味著過程單向流動而無迴圈。

使用它進行：

* 簡單 RAG：檢索文件，建立提示，從大語言模型獲得答案。
* 摘要：取用戶文字，送入摘要提示，並回傳輸出。
* 擷取：從文字區塊中擷取結構化資料 (如 JSON)。

Python

```python
# 一個簡單的 LCEL 鏈概念上 # (這不是可執行的程式碼，只是說明流程) 
chain = prompt | model | output_parse
```

## LangGraph

LangGraph 是建立在 LangChain 之上的函式庫，用於處理更進階的智能體系統。它允許你將工作流程定義為具有節點 (函數或 LCEL 鏈) 和邊 (條件邏輯) 的圖。其主要優勢是建立循環的能力，允許應用程式以靈活的順序迴圈、重試或呼叫工具，直到任務完成。它明確管理應用程式狀態，該狀態在節點之間傳遞並在整個過程中更新。

使用它進行：

* 多智能體系統：監督智能體將任務路由到專門的工作智能體，可能迴圈直到達成目標。
* 規劃和執行智能體：智能體建立計劃，執行步驟，然後迴圈返回以基於結果更新計劃。
* 人機協作：圖可以等待人工輸入，然後決定下一個節點。

| 功能 | LangChain | LangGraph |
| :---- | :---- | :---- |
| 核心抽象 | 鏈 (使用 LCEL) | 節點圖 |
| 工作流程類型 | 線性 (有向無環圖) | 循環 (具有迴圈的圖) |
| 狀態管理 | 每次執行通常無狀態 | 明確且持久的狀態物件 |
| 主要用途 | 簡單、可預測的序列 | 複雜、動態、有狀態的智能體 |

### 你應該使用哪一個？

* 當你的應用程式有清晰、可預測且線性的步驟流程時，選擇 LangChain。如果你可以定義從 A 到 B 到 C 的過程而無需迴圈返回，使用 LCEL 的 LangChain 是完美的工具。
* 當你需要應用程式推理、規劃或在迴圈中操作時，選擇 LangGraph。如果你的智能體需要使用工具、反思結果，並可能用不同方法再次嘗試，你需要 LangGraph 的循環和有狀態特性。

```python
# 圖狀態
class State(TypedDict):
    topic: str
    joke: str
    story: str
    poem: str
    combined_output: str


# 節點
def call_llm_1(state: State):
    """第一次大語言模型呼叫來產生初始笑話"""
    msg = llm.invoke(f"寫一個關於 {state['topic']} 的笑話")
    return {"joke": msg.content}


def call_llm_2(state: State):
    """第二次大語言模型呼叫來產生故事"""
    msg = llm.invoke(f"寫一個關於 {state['topic']} 的故事")
    return {"story": msg.content}


def call_llm_3(state: State):
    """第三次大語言模型呼叫來產生詩"""
    msg = llm.invoke(f"寫一首關於 {state['topic']} 的詩")
    return {"poem": msg.content}


def aggregator(state: State):
    """將笑話和故事結合為單一輸出"""
    combined = f"這裡是關於 {state['topic']} 的故事、笑話和詩！\n\n"
    combined += f"故事：\n{state['story']}\n\n"
    combined += f"笑話：\n{state['joke']}\n\n"
    combined += f"詩：\n{state['poem']}"
    return {"combined_output": combined}


# 建置工作流程
parallel_builder = StateGraph(State)

# 添加節點
parallel_builder.add_node("call_llm_1", call_llm_1)
parallel_builder.add_node("call_llm_2", call_llm_2)
parallel_builder.add_node("call_llm_3", call_llm_3)
parallel_builder.add_node("aggregator", aggregator)

# 添加邊來連接節點
parallel_builder.add_edge(START, "call_llm_1")
parallel_builder.add_edge(START, "call_llm_2")
parallel_builder.add_edge(START, "call_llm_3")
parallel_builder.add_edge("call_llm_1", "aggregator")
parallel_builder.add_edge("call_llm_2", "aggregator")
parallel_builder.add_edge("call_llm_3", "aggregator")
parallel_builder.add_edge("aggregator", END)

parallel_workflow = parallel_builder.compile()

# 顯示工作流程
display(Image(parallel_workflow.get_graph().draw_mermaid_png()))

# 調用
state = parallel_workflow.invoke({"topic": "cats"})
print(state["combined_output"])
```

這段程式碼定義並執行一個並行運作的 LangGraph 工作流程。其主要目的是同時產生關於給定主題的笑話、故事和詩，然後將它們結合為單一的格式化文字輸出。

## Google 的 ADK

Google 的智能體開發套件 (ADK) 提供了一個高階、結構化的框架，用於建置和部署由多個互動 AI 智能體組成的應用程式。它與 LangChain 和 LangGraph 形成對比，提供了一個更有主見且面向生產的系統來協調智能體協作，而不是為智能體的內部邏輯提供基本構建模組。

LangChain 在最基礎的層面運作，提供組件和標準化介面來建立操作序列，如呼叫模型並解析其輸出。LangGraph 通過引入更靈活且強大的控制流程來擴展這一點；它將智能體的工作流程視為有狀態的圖。使用 LangGraph，開發者明確定義節點 (函數或工具) 和邊 (決定執行路徑)。這種圖結構允許複雜的循環推理，其中系統可以迴圈、重試任務，並基於在節點之間傳遞的明確管理狀態物件做出決策。它給開發者對單一智能體思考過程的細緻控制或從第一原理構建多智能體系統的能力。

Google 的 ADK 抽象了大部分這種低階圖構建。它不要求開發者定義每個節點和邊，而是為多智能體互動提供預建的架構模式。例如，ADK 有內建的智能體類型，如 SequentialAgent 或 ParallelAgent，它們自動管理不同智能體之間的控制流程。它圍繞「團隊」智能體的概念架構，通常有一個主要智能體將任務委派給專門的子智能體。狀態和會話管理由框架更隱式地處理，提供比 LangGraph 明確狀態傳遞更具凝聚力但較不精細的方法。因此，雖然 LangGraph 給你詳細的工具來設計單一機器人或團隊的複雜配線，Google 的 ADK 給你一條工廠裝配線，設計來建置和管理一隊已經知道如何協同工作的機器人。

```python
from google.adk.agents import LlmAgent
from google.adk.tools import google_Search

dice_agent = LlmAgent(
    model="gemini-2.0-flash-exp",
    name="question_answer_agent",
    description="一個可以回答問題的有用助手智能體。",
    instruction="""使用 google 搜尋回應查詢""",
    tools=[google_search],
)
```

這段程式碼建立了一個搜尋增強的智能體。當這個智能體收到問題時，它不僅僅依賴其現有知識。相反，遵循其指令，它將使用 Google 搜尋工具從網路尋找相關的即時資訊，然後使用該資訊來構建其答案。

## Crew.AI

CrewAI 提供了一個編排框架，通過專注於協作角色和結構化過程來建置多智能體系統。它在比基礎工具套件更高的抽象層面運作，提供一個反映人類團隊的概念模型。開發者不定義邏輯的精細流程作為圖，而是定義角色和其分配，CrewAI 管理它們的互動。

這個框架的核心組件是智能體、任務和團隊。智能體不僅由其功能定義，還由角色定義，包括特定角色、目標和背景故事，這指導其行為和溝通風格。任務是具有清晰描述和預期輸出的離散工作單元，分配給特定智能體。團隊是包含智能體和任務清單的凝聚單元，它執行預定義的過程。這個過程決定工作流程，通常是順序的 (其中一個任務的輸出成為下一個任務的輸入) 或分層的 (其中類似經理的智能體委派任務並協調其他智能體之間的工作流程)。

與其他框架相比，CrewAI 佔據獨特位置。它遠離 LangGraph 的低階、明確狀態管理和控制流程，其中開發者配線每個節點和條件邊。開發者不建置狀態機，而是設計團隊章程。雖然 Google 的 ADK 為整個智能體生命週期提供全面、面向生產的平台，CrewAI 專門專注於智能體協作的邏輯和模擬專家團隊。

```python
@crew
def crew(self) -> Crew:
   """建立研究團隊"""
   return Crew(
     agents=self.agents,
     tasks=self.tasks,
     process=Process.sequential,
     verbose=True,
   )
```

這段程式碼為 AI 智能體團隊設定了順序工作流程，他們按特定順序處理任務清單，並啟用詳細日誌記錄來監控其進度。

## 其他智能體開發框架

**Microsoft AutoGen**：AutoGen 是一個專注於協調多個智能體透過對話解決任務的框架。其架構使具有不同能力的智能體能夠互動，允許複雜的問題分解和協作解決。AutoGen 的主要優勢是其靈活、對話驅動的方法，支援動態和複雜的多智能體互動。然而，這種對話範式可能導致較不可預測的執行路徑，並可能需要複雜的提示工程來確保任務有效收斂。

**LlamaIndex**：LlamaIndex 基本上是一個資料框架，設計用於將大語言模型與外部和私有資料來源連接。它擅長建立複雜的資料攝取和檢索管線，這對於建置能夠執行 RAG 的知識型智能體至關重要。雖然其資料索引和查詢能力對於建立脈絡感知智能體極其強大，但與智能體優先框架相比，其用於複雜智能體控制流程和多智能體編排的原生工具較不發達。當核心技術挑戰是資料檢索和綜合時，LlamaIndex 是最佳選擇。

**Haystack**：Haystack 是一個開源框架，專為建置由大語言模型驅動的可擴展且生產就緒的搜尋系統而設計。其架構由模組化、可互操作的節點組成，形成文件檢索、問答和摘要的管線。Haystack 的主要優勢是專注於大規模資訊檢索任務的效能和可擴展性，使其適用於企業級應用程式。潛在的權衡是其為搜尋管線優化的設計，對於實施高度動態和創意的智能體行為可能更加僵化。

**MetaGPT**：MetaGPT 通過基於一組預定義的標準作業程序 (SOPs) 分配角色和任務來實施多智能體系統。這個框架結構化智能體協作以模仿軟體開發公司，智能體承擔產品經理或工程師等角色來完成複雜任務。這種 SOP 驅動的方法產生高度結構化且連貫的輸出，這對於程式碼生成等專門領域是重要優勢。該框架的主要限制是其高度專業化，使其對於其核心設計之外的通用智能體任務較不適應。

**SuperAGI**：SuperAGI 是一個開源框架，設計為提供自主智能體的完整生命週期管理系統。它包括智能體供應、監控和圖形介面功能，旨在增強智能體執行的可靠性。關鍵好處是專注於生產就緒性，具有內建機制來處理迴圈等常見故障模式，並提供智能體效能的可觀察性。潛在缺點是其全面的平台方法可能引入比更輕量、基於函式庫的框架更多的複雜性和開銷。

**Semantic Kernel**：由 Microsoft 開發，Semantic Kernel 是一個 SDK，通過「外掛」和「規劃器」系統將大語言模型與傳統程式碼整合。它允許大語言模型調用原生函數並編排工作流程，有效地將模型視為更大軟體應用程式中的推理引擎。其主要優勢是與現有企業程式碼庫的無縫整合，特別是在 .NET 和 Python 環境中。與更直接的智能體框架相比，其外掛和規劃器架構的概念開銷可能呈現更陡峭的學習曲線。

**Strands Agents：** 一個 AWS 輕量且靈活的 SDK，使用模型驅動方法來建置和執行 AI 智能體。它被設計為簡單且可擴展，支援從基本對話助手到複雜多智能體自主系統的一切。該框架是模型無關的，為各種大語言模型提供者提供廣泛支援，並包括與 MCP 的原生整合以便輕鬆存取外部工具。其核心優勢是其簡單性和靈活性，具有易於開始使用的可自訂智能體迴圈。潛在的權衡是其輕量設計意味著開發者可能需要建置更多周圍的操作基礎設施，如進階監控或生命週期管理系統，這些更全面的框架可能提供現成的功能。

## 結論

智能體框架的景觀提供了多樣的工具光譜，從用於定義智能體邏輯的低階函式庫到用於編排多智能體協作的高階平台。在基礎層面，LangChain 實現簡單、線性的工作流程，而 LangGraph 引入了有狀態、循環的圖以進行更複雜的推理。像 CrewAI 和 Google 的 ADK 等更高階框架將焦點轉移到編排具有預定義角色的智能體團隊，而像 LlamaIndex 等其他框架則專精於資料密集型應用程式。這種多樣性為開發者呈現了基於圖的系統精細控制與更有主見平台的簡化開發之間的核心權衡。因此，選擇正確的框架取決於應用程式是否需要簡單序列、動態推理迴圈或管理的專家團隊。最終，這個不斷演進的生態系統通過讓開發者選擇其專案需求的精確抽象層次，賦予他們建置越來越複雜的 AI 系統的能力。

參考文獻

1. LangChain, [https://www.langchain.com/](https://www.langchain.com/)
2. LangGraph, [https://www.langchain.com/langgraph](https://www.langchain.com/langgraph)
3. Google's ADK, [https://google.github.io/adk-docs/](https://google.github.io/adk-docs/)
4. Crew.AI, [https://docs.crewai.com/en/introduction](https://docs.crewai.com/en/introduction)