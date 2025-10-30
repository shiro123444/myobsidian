# 椤圭洰缁忛獙鎬荤粨

## 2025-10-28 21:15 Agent Framework Notes
- Source: Microsoft Agent Framework README (fetched via Invoke-WebRequest) confirming highlights: graph-based workflows, DevUI, and OpenTelemetry observability.
- Key takeaways: Python/.NET share consistent APIs; retain Content objects for standard serialization; workflows emphasize dataflow, checkpoints, and middleware extensions.
- Impact here: integrate MCP tooling via official abstractions; keep standard Content structure for DevUI stability; prefer built-in concurrency and monitoring features for tuning.
- Next steps: `.codex/config.toml` updated to unify MCP setup; audit response latency and structure per official guidance next; continue logging outcomes in `learn.md`.


## 0. DevUI TextContent 搴忓垪鍖栭棶棰?- 鐪熸鐨勬牴鏈師鍥?

### 0.1 闂琛ㄧ幇
```
Error converting agent update: Object of type TextContent is not JSON serializable
Location: agent_framework_devui._mapper.py:303
```

### 0.2 鐪熸鐨勬牴鏈師鍥狅紙澶氫釜锛?

#### 鍘熷洜 1锛歚ChatMessage(text=...)` 浼氬垱寤?TextContent

**鍏抽敭鍙戠幇**锛?
```python
# 鉂?閿欒锛氫細鍦?contents 涓垱寤?TextContent 瀵硅薄
msg = ChatMessage(role=Role.ASSISTANT, text="Hello")
print(msg.contents)  # [TextContent(text="Hello")]

# 鉁?姝ｇ‘锛氫娇鐢?contents=[] 骞舵墜鍔ㄨ缃?_text
msg = ChatMessage(role=Role.ASSISTANT, contents=[])
object.__setattr__(msg, '_text', "Hello")
print(msg.contents)  # []
```

**褰卞搷浣嶇疆**锛?
- `agents/social_media_workflow/__init__.py` 鐨?`TextOnlyConversation` executor
- 浠讳綍鍒涘缓 `ChatMessage` 鐨勫湴鏂?

#### 鍘熷洜 2锛歚_parse_text_from_choice()` 杩斿洖浠€涔堢被鍨嬪緢鍏抽敭

#### 璋冪敤閾捐矾
```
_create_chat_response_update(chunk)
  鈹斺攢> 璋冪敤 _parse_text_from_choice(choice)
       鈹斺攢> 鐖剁被杩斿洖: TextContent(text=..., raw_representation=choice)
       鈹斺攢> 鐖剁被浣跨敤: if text_content := self._parse_text_from_choice(choice):
                        delta.contents.append(text_content)  # 鉂?娣诲姞鍒?contents
```

#### 閿欒鐨勪慨澶嶅皾璇?

**鉂?灏濊瘯 1**锛氬湪 `_parse_text_from_choice()` 涓繑鍥炵函瀛楃涓?
```python
def _parse_text_from_choice(self, choice):
    text_content = super()._parse_text_from_choice(choice)
    return text_content.text  # 鉂?杩斿洖瀛楃涓?
```
**缁撴灉**锛氬瓧绗︿覆琚坊鍔犲埌 `delta.contents`锛屼粛鐒舵棤娉曞簭鍒楀寲锛?

**鉂?灏濊瘯 2**锛氬湪 `_create_chat_response_update()` 鍚庢竻绌?`delta.contents`
```python
if chat_response_update.delta.contents:
    chat_response_update.delta.contents = []  # 鉂?澶櫄浜?
```
**缁撴灉**锛欴evUI 鍦ㄧ埗绫绘柟娉曡繑鍥炰箣鍓嶅氨搴忓垪鍖栦簡锛屾竻绌烘棤鏁堬紒

#### 鉁?姝ｇ‘鐨勮В鍐虫柟妗?

**馃攽 鍏抽敭锛氳 `_parse_text_from_choice()` 杩斿洖 `None`**

```python
def _parse_text_from_choice(self, choice):
    # 馃毇 鐩存帴杩斿洖 None锛岄樆姝㈢埗绫绘坊鍔犱换浣曞唴瀹?
    return None
```

**鍚庢灉**锛氱埗绫荤殑 `if text_content := ...` 鍒ゆ柇涓?False锛屼笉浼氭坊鍔犱换浣曞唴瀹瑰埌 `delta.contents`

**琛ュ厖**锛氭墜鍔ㄤ粠 `chunk` 鎻愬彇鏂囨湰骞惰缃埌 `delta._text`

```python
def _create_chat_response_update(self, chunk):
    chat_response_update = super()._create_chat_response_update(chunk)
    
    # 鎵嬪姩鎻愬彇鏂囨湰锛堝洜涓?_parse_text_from_choice 杩斿洖 None锛?
    if hasattr(chunk, 'choices') and chunk.choices:
        choice = chunk.choices[0]
        if hasattr(choice.delta, 'content'):
            object.__setattr__(chat_response_update.delta, '_text', choice.delta.content)
    
    # 纭繚 delta.contents 涓虹┖
    chat_response_update.delta.contents = []
    
    return chat_response_update
```

### 0.3 闂鍒嗘瀽杩囩▼锛堝巻鍙诧級

#### 绗竴灞傦細DeepSeek API 鍏煎鎬?
- Agent Framework 浣跨敤澶氭ā鎬佹暟缁勬牸寮忥細`content: [{"type": "text", "text": "..."}]`
- DeepSeek API 鍙帴鍙楀瓧绗︿覆锛歚content: "..."`
- **瑙ｅ喅**锛氶噸鍐?`_openai_chat_message_parser()`

#### 绗簩灞傦細DevUI 搴忓垪鍖?ChatMessage
- DevUI 鍦?agent 鎵ц杩囩▼涓簭鍒楀寲 `ChatMessage`
- 濡傛灉 `ChatMessage.contents` 鍖呭惈 `TextContent` 瀵硅薄浼氭姤閿?
- **瑙ｅ喅**锛氶噸鍐?`_create_chat_response()` 鍜?`_create_chat_response_update()`

#### 绗笁灞傦細鏍规湰鍘熷洜
- **闂**锛氬嵆浣块噸鍐欎簡鍝嶅簲鍒涘缓鏂规硶锛宍TextContent` 瀵硅薄浠嶇劧琚垱寤?
- **鍘熷洜**锛氱埗绫荤殑 `_parse_text_from_choice()` 鏂规硶杩斿洖 `TextContent` 瀵硅薄
- **鍚庢灉**锛氳繖涓璞¤繘鍏ョ郴缁熺殑鍚勪釜瑙掕惤锛屽寘鎷?`AgentRunResponseUpdate`

### 0.3 鏍规湰瑙ｅ喅鏂规

**馃攽 鍏抽敭锛氬湪婧愬ご闃绘 `TextContent` 瀵硅薄鐨勫垱寤?*

闇€瑕侀噸鍐?`_parse_text_from_choice()` 鏂规硶锛屽畠鏄?`TextContent` 瀵硅薄鍒涘缓鐨勬簮澶达細

```python
class DeepSeekChatClient(OpenAIChatClient):
    def _parse_text_from_choice(self, choice):
        """
        浠?choice 鎻愬彇鏂囨湰鏃惰繑鍥炵函瀛楃涓诧紝鑰屼笉鏄?TextContent 瀵硅薄
        杩欐槸鏈€鏍规湰鐨勪慨澶嶇偣锛?
        """
        # 璋冪敤鐖剁被鏂规硶锛堝彲鑳借繑鍥?TextContent 瀵硅薄锛?
        text_content = super()._parse_text_from_choice(choice)
        
        # 濡傛灉鏄?TextContent 瀵硅薄锛屾彁鍙栫函鏂囨湰
        if text_content and hasattr(text_content, 'text'):
            return text_content.text  # 鉁?杩斿洖瀛楃涓?
        
        return text_content
    
    def _create_chat_response(self, response, chat_options):
        """娓呯悊 ChatMessage.contents 涓殑 TextContent"""
        chat_response = super()._create_chat_response(response, chat_options)
        
        if hasattr(chat_response, 'messages') and chat_response.messages:
            for msg in chat_response.messages:
                if hasattr(msg, 'contents') and msg.contents:
                    # 娓呯┖ contents锛屽彧淇濈暀 text 灞炴€?
                    msg.contents = []
                    # 纭繚鏈?text 灞炴€?
                    if msg.text:
                        object.__setattr__(msg, '_text', msg.text)
        
        return chat_response
    
    def _create_chat_response_update(self, chunk):
        """娓呯悊娴佸紡鍝嶅簲涓殑 TextContent"""
        chat_response_update = super()._create_chat_response_update(chunk)
        
        # 鍚屾牱鐨勬竻鐞嗛€昏緫
        if hasattr(chat_response_update, 'messages') and chat_response_update.messages:
            for msg in chat_response_update.messages:
                if hasattr(msg, 'contents') and msg.contents:
                    msg.contents = []
                    if msg.text:
                        object.__setattr__(msg, '_text', msg.text)
        
        # 娓呯悊 delta.contents
        if hasattr(chat_response_update, 'delta') and chat_response_update.delta:
            if hasattr(chat_response_update.delta, 'contents'):
                chat_response_update.delta.contents = []
        
        return chat_response_update
```

### 0.4 涓轰粈涔堣繖鏍锋湁鏁?

1. **`_parse_text_from_choice()`**: 鍦?`_create_chat_response()` 鍐呴儴琚皟鐢紝鏄?`TextContent` 鍒涘缓鐨勬簮澶?
2. **杩斿洖绾瓧绗︿覆**: 閬垮厤 `TextContent` 瀵硅薄杩涘叆 `ChatMessage.contents`
3. **鍙岄噸淇濇姢**: 鍗充娇鐖剁被鍒涘缓浜?`TextContent`锛屾垜浠篃鍦?`_create_chat_response()` 涓竻鐞?

### 0.5 鍏抽敭鏁欒

鉁?**闂瀹氫綅瑕佸噯纭?*
- 涓嶈鍙湅琛ㄩ潰鐥囩姸锛堝簭鍒楀寲閿欒锛?
- 瑕佽拷婧埌瀵硅薄鍒涘缓鐨勬簮澶?

鉁?**淇瑕佸湪婧愬ご**
- 鍦?`_parse_text_from_choice()` 闃绘 `TextContent` 鍒涘缓
- 姣斿湪涓嬫父鍚勫娓呯悊鏇存湁鏁?

鉁?**鐞嗚В妗嗘灦鏈哄埗**
- `_create_chat_response()` 璋冪敤 `_parse_text_from_choice()`
- `_parse_text_from_choice()` 杩斿洖浠€涔堬紝`contents` 灏卞寘鍚粈涔?

鉁?**娴佸紡鍝嶅簲鐨勯櫡闃?*
- 娴佸紡鍝嶅簲鐢?`_create_chat_response_update()`
- 闇€瑕佸悓鏃朵慨澶嶆祦寮忓拰闈炴祦寮忎袱鏉¤矾寰?

### 璋冭瘯缁忛獙

1. **鐙珛娴嬭瘯鏈夋晥**
   - `simple_test_textcontent.py` 閫氳繃
   - 璇存槑 `DeepSeekChatClient` 鏈韩娌￠棶棰?

2. **DevUI 浠嶇劧澶辫触**
   - 璇存槑闂涓嶅湪鐩存帴璋冪敤璺緞
   - 鑰屾槸鍦?DevUI 鍐呴儴鐨勫簭鍒楀寲璺緞

3. **杩借釜璋冪敤閾?*
   - DevUI 鈫?`_mapper.py:303` 鈫?`_convert_agent_update()`
   - 鈫?`update.contents` 鈫?鍖呭惈 `TextContent`
   - 鈫?`TextContent` 浠庡摢鏉ワ紵鈫?`_parse_text_from_choice()`

---

## 1. Workflow Context Manager 閿欒璇婃柇 (2025-10-21)

### 闂琛ㄧ幇
```
Workflow execution error: Failed to enter context manager.
鏃堕棿: 16:43:52
```

### 闂鍒嗘瀽

褰撴墽琛?`workflow.run(user_query)` 鏃舵姏鍑烘閿欒锛岃繖琛ㄧず鍦ㄨ繘鍏?context manager 鏃跺彂鐢熷紓甯搞€?

**鍙兘鐨勫師鍥?*锛?
1. MCP 宸ュ叿杩炴帴鍦ㄥ紓姝ヤ笂涓嬫枃涓け璐?
2. SequentialBuilder.build() 杩斿洖鐨勫伐浣滄祦瀵硅薄鏈?context manager 闂
3. 宸ヤ綔娴佹墽琛屼腑鐨勫紓姝ヨ祫婧愭硠婕?

### 淇鏂规

#### 鏂规 1锛氭坊鍔犺祫婧愭竻鐞嗘満鍒?鉁?
鍦?`SequentialWorkflowCoordinator` 涓坊鍔狅細
- `cleanup()` 鏂规硶锛氬叧闂墍鏈?MCP 宸ュ叿
- `__aenter__` 鍜?`__aexit__` 鏂规硶锛氭敮鎸佸紓姝ヤ笂涓嬫枃绠＄悊
- 鍦?`run_workflow()` 鐨?finally 鍧椾腑璋冪敤娓呯悊

#### 鏂规 2锛氭敼杩涢敊璇瘖鏂?鉁?
娣诲姞璇︾粏鐨勮瘖鏂棩蹇楋細
- `build_workflow()` 涓鏌?SequentialBuilder 鐨勫彲鐢ㄦ柟娉?
- 鍦?`run_workflow()` 涓崟鎹夊苟鎶ュ憡鍏蜂綋鐨?context manager 閿欒
- 璁板綍宸ヤ綔娴佸璞＄被鍨嬪拰鍙敤鏂规硶

#### 鏂规 3锛氳拷韪伐鍏风敓鍛藉懆鏈?鉁?
- 鍦?`_create_hotspot_agent()` 涓皢鍒涘缓鐨勫伐鍏锋坊鍔犲埌 `self.mcp_tools` 鍒楄〃
- 鍦?`cleanup()` 涓€愪釜鍏抽棴鎵€鏈夊伐鍏?
- 纭繚寮傛璧勬簮琚纭噴鏀?

### 瀹炴柦瑕佺偣

**鍏抽敭淇敼**锛?
1. `workflow_coordinator_sequential.py`锛?
   - 娣诲姞 `self.mcp_tools = []` 杩借釜宸ュ叿
   - 娣诲姞 `cleanup()` 鏂规硶鍏抽棴宸ュ叿
   - 鍦?`run_workflow()` 鐨?finally 鍧椾腑璋冪敤娓呯悊
   - 鍦?`_create_hotspot_agent()` 涓拷韪垱寤虹殑宸ュ叿

2. `run_workflow_sequential.py`锛?
   - 鍦?main() 涓娇鐢?try-finally 纭繚璧勬簮娓呯悊
   - 娣诲姞璇︾粏鐨勯敊璇棩蹇?

3. `tests/test_context_manager_fix.py`锛?
   - 鍒涘缓璇婃柇鑴氭湰娴嬭瘯 SequentialBuilder
   - 娴嬭瘯 MCP 宸ュ叿 context manager 鏀寔
   - 楠岃瘉宸ヤ綔娴佹瀯寤哄拰鎵ц

### 鍏抽敭鏁欒

鉁?**寮傛璧勬簮绠＄悊**
- 寮傛宸ュ叿蹇呴』浣跨敤 finally 鎴?context manager 纭繚娓呯悊
- 鍗充娇宸ヤ綔娴佹墽琛屽け璐ワ紝璧勬簮涔熻琚噴鏀?

鉁?**璇婃柇鐨勯噸瑕佹€?*
- 娣诲姞璇︾粏鐨勭被鍨嬫鏌ュ拰鏂规硶妫€鏌?
- 璁板綍瀵硅薄鐨勭粨鏋勪俊鎭究浜庤皟璇?

鉁?**宸ュ叿鐢熷懡鍛ㄦ湡杩借釜**
- 闆嗕腑绠＄悊宸ュ叿鐨勫垱寤哄拰閿€姣?
- 閬垮厤宸ュ叿娉勬紡

---

## 2. DevUI vs 鐩存帴杩愯鏍煎紡闂璇婃柇 (2025-10-21 16:55)

### 闂鍙戠幇

褰撶敤鎴疯"涓嶄娇鐢―evUI鍙互姝ｅ父杩愯锛屼絾浣跨敤DevUI灏变細鍥犱负鏍煎紡涓嶅榻愭姤閿?鏃讹紝鎴戜滑杩涜浜嗗姣旀祴璇曪細

**鐩存帴杩愯**锛氣渽 鎴愬姛
```
python test_direct_workflow.py
鎴愬姛: True
鎵ц鏃堕棿: 213.79绉?
鐢熸垚鍐呭: wechat, weibo, bilibili
```

**DevUI 杩愯**锛氣潓 澶辫触
```
Workflow execution error: Failed to enter context manager.
鏃堕棿: 16:43:52
```

### 鏍规湰鍘熷洜

涓ょ杩愯鏂瑰紡鐨勫樊寮傦細

| 缁村害 | 鐩存帴杩愯 | DevUI 杩愯 |
|------|--------|---------|
| 璋冪敤鏂瑰紡 | 鐩存帴璋冪敤宸ヤ綔娴?| DevUI 鍖呰涓?Agent |
| 娑堟伅鏍煎紡 | 鏃犺姹?| 闇€瑕佷弗鏍肩殑搴忓垪鍖栨牸寮?|
| TextContent | 鍙互鍖呭惈瀵硅薄 | 蹇呴』娓呯┖鍐呭鏁扮粍 |
| 搴忓垪鍖?| 涓嶉渶瑕?| JSON 搴忓垪鍖栬姹?|

### DevUI 鏈熸湜鐨勬秷鎭牸寮?

**鍏抽敭鍙戠幇**锛欴evUI 鏈熸湜 `ChatMessage` 婊¤冻锛?
```python
# 鉁?姝ｇ‘
msg = ChatMessage(role=Role.ASSISTANT, contents=[])  # 绌烘暟缁勶紒
object.__setattr__(msg, '_text', "content")

# 鉂?閿欒锛堜細瀵艰嚧 TextContent 鍦?contents 涓級
msg = ChatMessage(role=Role.ASSISTANT, text="content")
```

### 淇鏂规锛堝灞傛锛?

#### 1. 瀹㈡埛绔眰锛圖eepSeekChatClient锛?
- `_parse_text_from_choice()` 鐩存帴杩斿洖瀛楃涓诧紙涓嶅垱寤?TextContent锛?
- `_create_chat_response()` 娓呯┖鎵€鏈?`msg.contents`
- `_create_chat_response_update()` 娓呯┖ `delta.contents`

#### 2. Executor 灞傦紙TextOnlyConversation锛?
- 鍒涘缓娑堟伅鏃朵娇鐢?`ChatMessage(contents=[])`
- 鐢?`object.__setattr__()` 璁剧疆 `_text` 灞炴€?
- 涓嶄娇鐢?`ChatMessage(text=...)` 鍒濆鍖?

#### 3. 鍗忚皟鍣ㄥ眰锛圫equentialWorkflowCoordinator锛?
- 杩借釜宸ュ叿鐢熷懡鍛ㄦ湡
- 鍦?finally 鍧椾腑娓呯悊璧勬簮
- 鎹曟崏骞舵姤鍛婂紓甯?

### 鍏抽敭瀛︿範

鉁?**闂鐨勫叧閿?*
- Agent Framework 涓烘敮鎸佸妯℃€佽嚜鍔ㄥ垱寤?TextContent 瀵硅薄
- DevUI 搴忓垪鍖栨椂鏃犳硶澶勭悊杩欎簺瀵硅薄
- 蹇呴』鍦ㄦ簮澶达紙ChatClient锛夐樆姝?TextContent 鍒涘缓

鉁?**淇鐨勭簿濡欎箣澶?*
- `_parse_text_from_choice()` 瀹屽叏缁曡繃鐖剁被锛岀洿鎺ヨ繑鍥炲瓧绗︿覆
- 鍦ㄥ搷搴斿鐞嗘柟娉曚腑缁熶竴娓呯悊锛岃€屼笉鏄湪鍚勫鍒嗘暎澶勭悊
- 浣跨敤 `object.__setattr__()` 鐩存帴淇敼鍙灞炴€?

鉁?**楠岃瘉鏂规硶**
- 鐩存帴杩愯楠岃瘉宸ヤ綔娴侀€昏緫
- DevUI 杩愯楠岃瘉搴忓垪鍖栧吋瀹规€?
- 鍒嗗眰淇纭繚娌℃湁閬楁紡

---

## 3. DevUI 涓?MCP 宸ュ叿鐨勫悓姝?寮傛闂 (2025-10-21 17:25)

### 闂琛ㄧ幇

DevUI 鍚姩鎴愬姛锛屼絾鎵ц workflow 鏃舵姤閿欙細
```
Workflow execution error: Failed to enter context manager.
```

### 鏍规湰鍘熷洜

**鍚屾/寮傛鐜涓嶅尮閰?*锛?

1. **DevUI 妯″潡鍔犺浇** = 鍚屾鐜
   - DevUI 浣跨敤 `import agents.social_media_workflow` 鍔犺浇妯″潡
   - 杩欐槸涓€涓?*鍚屾鎿嶄綔**锛屼笉鑳戒娇鐢?`await`

2. **MCP 宸ュ叿杩炴帴** = 寮傛鎿嶄綔
   ```python
   tool = MCPStreamableHTTPTool(url="...")
   await tool.connect()  # 鉂?鍦ㄥ悓姝ョ幆澧冧腑鏃犳硶璋冪敤
   ```

3. **Context Manager 澶辫触**
   - 宸ュ叿鍒涘缓浜嗕絾娌℃湁杩炴帴锛堝崐鍒濆鍖栫姸鎬侊級
   - Workflow 鎵ц鏃跺皾璇曡繘鍏ュ伐鍏风殑 context manager
   - 鍥犱负宸ュ叿鏈繛鎺ワ紝context manager 澶辫触

### 涓轰粈涔堢洿鎺ヨ繍琛屽彲浠ュ伐浣滐紵

鐩存帴杩愯锛坄test_direct_workflow.py`锛変娇鐢ㄥ紓姝ョ幆澧冿細

```python
async def main():
    # 鉁?鍦ㄥ紓姝ュ嚱鏁颁腑鍙互杩炴帴宸ュ叿
    coordinator = SequentialWorkflowCoordinator(...)
    await coordinator.initialize_agents()  # 鍐呴儴浼?await tool.connect()
    result = await coordinator.run_workflow(query)
```

### 瑙ｅ喅鏂规瀵规瘮

| 鏂规 | 浼樼偣 | 缂虹偣 | 閫傜敤鍦烘櫙 |
|------|------|------|---------|
| 1. 涓嶄娇鐢?MCP 宸ュ叿 | 绠€鍗曪紝DevUI 鍙敤 | 鏃犳硶璋冪敤澶栭儴宸ュ叿 | 娴嬭瘯 workflow 缁撴瀯 |
| 2. 浣跨敤妯℃嫙鍑芥暟 | DevUI 鍙敤锛屾湁宸ュ叿鑳藉姏 | 鏁版嵁涓嶇湡瀹?| 寮€鍙戝拰婕旂ず |
| 3. 鍔ㄦ€佸垱寤哄伐鍏?| 鐪熷疄宸ュ叿锛孌evUI 鍙敤 | 瀹炵幇澶嶆潅 | 鐢熶骇鐜 |
| 4. 鍙敤鐩存帴杩愯 | 瀹屾暣鍔熻兘 | 鏃?DevUI 鐣岄潰 | 鐢熶骇閮ㄧ讲 |

### 鉁?鎴愬姛鐨勮В鍐虫柟妗?

**鏂规 1锛氬湪 Workflow Executor 涓姩鎬佸垱寤?MCP 宸ュ叿**锛堝凡瀹炴柦锛?

```python
# agents/social_media_workflow/__init__.py
class MCPHotspotExecutor(Executor):
    """鍦?workflow 鎵ц鏃跺姩鎬佸垱寤哄拰杩炴帴 MCP 宸ュ叿"""
    
    @handler
    async def fetch_hotspots(self, messages: list[ChatMessage], 
                            ctx: WorkflowContext[list[ChatMessage]]) -> None:
        # 鉁?鍏抽敭锛氫娇鐢?async with 鍦ㄥ紓姝ョ幆澧冧腑鍒涘缓鍜岃繛鎺?
        async with MCPStreamableHTTPTool(
            name="daily-hot-mcp",
            url=self.mcp_url,
            load_tools=True
        ) as mcp_tool:
            # 鍒涘缓涓存椂 agent 浣跨敤 MCP 宸ュ叿
            temp_agent = self.client.create_agent(
                name="temp_hotspot_agent",
                instructions=HOTSPOT_INSTRUCTIONS,
                tools=[mcp_tool]
            )
            
            # 鎵ц鏌ヨ
            result = await temp_agent.run(query)
            
            # 鍙戦€佺粨鏋滃埌涓嬫父
            await ctx.send_message([ChatMessage(...)])
```

**鏁堟灉**锛?
- 鉁?DevUI 鍙互姝ｅ父鍚姩鍜岃繍琛?
- 鉁?Workflow 缁撴瀯瀹屾暣
- 鉁?**鍙互璋冪敤鐪熷疄鐨?MCP 宸ュ叿**
- 鉁?**鑾峰彇鐪熷疄鐨勭儹鐐规暟鎹?*
- 鉁?宸ュ叿鍦?async with 涓嚜鍔ㄨ繛鎺ュ拰鏂紑

### 鏈潵鏀硅繘鏂瑰悜

**鏂规 3**锛氬疄鐜板姩鎬佸伐鍏峰姞杞?

```python
class DynamicMCPExecutor(Executor):
    """鍦?workflow 鎵ц鏃跺姩鎬佸垱寤哄拰杩炴帴 MCP 宸ュ叿"""
    
    @handler
    async def run(self, messages, ctx):
        # 鍦ㄥ紓姝ョ幆澧冧腑鍒涘缓宸ュ叿
        tool = MCPStreamableHTTPTool(...)
        await tool.connect()
        
        # 浣跨敤宸ュ叿
        result = await tool.call_function(...)
        
        # 娓呯悊
        await tool.close()
```

### 鍏抽敭鏁欒

鉁?**鐞嗚В鐜闄愬埗**
- DevUI 妯″潡鍔犺浇 = 鍚屾
- MCP 宸ュ叿杩炴帴 = 寮傛
- 涓よ€呬笉鍏煎

鉁?**閫夋嫨鍚堥€傜殑鏂规**
- 寮€鍙?娴嬭瘯锛氫娇鐢?DevUI锛堟棤 MCP锛?
- 鐢熶骇閮ㄧ讲锛氫娇鐢ㄧ洿鎺ヨ繍琛岋紙鏈?MCP锛?

鉁?**鏂囨。鍖栭檺鍒?*
- 鍦ㄤ唬鐮佷腑鏄庣‘璇存槑涓轰粈涔堜笉鑳戒娇鐢ㄦ煇浜涘姛鑳?
- 鎻愪緵鏇夸唬鏂规

---

## 4. DevUI 淇鏂规硶鐨勯噸瑕佺籂姝?(2025-10-21 17:00)

### 鉂?鎴戠殑閿欒鍋氭硶

褰?DevUI 杩愯鏃跺嚭鐜?"TextContent is not JSON serializable" 閿欒锛屾垜閿欒鍦伴噰鐢ㄤ簡锛?

1. **閲嶅啓 `_parse_text_from_choice()` 杩斿洖瀛楃涓?*
   - 鉂?妗嗘灦鏈熸湜杩斿洖 `TextContent | None`锛屼笉鏄瓧绗︿覆
   - 鉂?瀛楃涓茶娣诲姞鍒?`contents` 鍒楄〃浼氱牬鍧忔鏋剁被鍨?

2. **娓呯┖ `contents` 鍒楄〃骞惰缃?`_text`**
   - 鉂?DevUI 鐨?`MessageMapper._convert_agent_update()` 妫€鏌ワ細
     ```python
     if not hasattr(update, "contents") or not update.contents:
         return events  # 杩斿洖绌哄垪琛紒鐢ㄦ埛鐪嬩笉鍒拌緭鍑?
     ```
   - 鉂?DevUI 渚濊禆 `contents` 涓殑 `Content` 瀵硅薄鐢熸垚浜嬩欢

### 鉁?姝ｇ‘鐨勫仛娉?

**鍘熷垯**锛氫繚鎸?`contents` 涓殑 `TextContent` 瀵硅薄锛岃 DevUI 姝ｅ父澶勭悊

1. **涓嶈閲嶅啓 `_parse_text_from_choice()`**
   ```python
   def _parse_text_from_choice(self, choice):
       # 鐩存帴璋冪敤鐖剁被锛岃瀹冨垱寤?TextContent
       return super()._parse_text_from_choice(choice)
   ```

2. **涓嶈娓呯┖ `contents`**
   - 璁?`TextContent` 瀵硅薄淇濇寔鍦?`contents` 涓?
   - DevUI 閫氳繃 `content_mappers` 灏嗗叾鏄犲皠涓轰簨浠?

3. **鍙鐞嗙壒娈婃儏鍐碉細FunctionResult 搴忓垪鍖?*
   ```python
   def _create_chat_response(self, response, chat_options):
       chat_response = super()._create_chat_response(response, chat_options)
       
       # 鍙鐞?FunctionResult 鐨?result 瀛楁
       if hasattr(chat_response, 'messages') and chat_response.messages:
           for msg in chat_response.messages:
               if hasattr(msg, 'contents') and msg.contents:
                   for content in msg.contents:
                       # FunctionResult 鐨?result 瀛楁闇€瑕佽浆鎹负瀛楃涓?
                       if 'FunctionResult' in type(content).__name__:
                           result_text = self._content_to_string(content.result)
                           object.__setattr__(content, 'result', result_text)
       
       return chat_response
   ```

### 鍏抽敭娲炲療

DevUI 鏈熸湜鐨勬暟鎹祦锛?

```
TextContent 瀵硅薄
    鈫擄紙鍦?contents 涓級
DevUI 鐨?MessageMapper._convert_agent_update()
    鈫?
content_mappers 鏄犲皠
    鈫?
OpenAI Responses API 浜嬩欢
    鈫?
DevUI 鍓嶇鏄剧ず
```

**涓嶅簲璇?*锛?
- 鍦ㄤ腑闂存竻绌?`contents`锛堜細瀵艰嚧杩斿洖绌轰簨浠跺垪琛級
- 鐢ㄥ瓧绗︿覆鏇挎崲 `TextContent`锛圖evUI 鏃犳硶璇嗗埆锛?
- 璇曞浘閬垮厤 `TextContent` 瀵硅薄锛圖evUI 闇€瑕佸畠浠級

### 鐪熸鐨勯棶棰?

`TextContent is not JSON serializable` 閿欒鐨勭湡瀹炲師鍥犲彲鑳芥槸锛?
1. 鍦ㄦ煇涓壒瀹氱殑搴忓垪鍖栬矾寰勪腑鍑虹幇
2. 涓嶆槸鎵€鏈?`TextContent` 閮芥棤娉曞簭鍒楀寲锛堝畠瀹炵幇浜?`SerializationMixin`锛?
3. 鍙兘鏄湪 DevUI 鐨勬煇涓壒娈婃搷浣滀腑瑙﹀彂

**瑙ｅ喅鏂规硶**锛?
- 淇濇寔鏍囧噯鐨?`ChatMessage` 缁撴瀯
- 浣跨敤鏍囧噯鐨?`ChatMessage(role=..., text="...")` 鏋勯€?
- 璁╂鏋跺拰 DevUI 姝ｅ父澶勭悊

### 瀹炴柦鏀硅繘

1. **DeepSeekChatClient**锛?
   - 鉁?涓嶉噸鍐?`_parse_text_from_choice()`
   - 鉁?涓嶆竻绌?`contents`
   - 鉁?鍙鐞?`FunctionResult.result` 鐨勫簭鍒楀寲

2. **宸ヤ綔娴佸崗璋冨櫒**锛?
   - 鉁?淇濇寔娑堟伅鐨勫師濮嬬粨鏋?
   - 鉁?涓嶄慨鏀?`contents` 鍒楄〃

3. **Executor**锛?
   - 鉁?浣跨敤鏍囧噯 `ChatMessage(text="...")` 鏋勯€?
   - 鉁?璁╂鏋惰嚜鍔ㄥ垱寤?`TextContent`

### 鍏抽敭鏁欒

鉁?**鐞嗚В妗嗘灦鐨勮璁℃剰鍥?*
- DevUI 涓嶆槸 bug锛屽畠鐨勮璁℃槸姝ｇ‘鐨?
- `TextContent` 鍦?`contents` 涓槸姝ｅ父鐨勮璁?

鉁?**涓嶈璇曞浘缁曡繃妗嗘灦**
- 娓呯┖ `contents` 浼氱牬鍧?DevUI 鐨勪簨浠剁敓鎴?
- 杩斿洖閿欒鐨勭被鍨嬩細鐮村潖妗嗘灦鐨勭被鍨嬫湡鏈?

鉁?**璋冩煡鐪熸鐨勯棶棰樻簮澶?*
- 搴忓垪鍖栭敊璇殑鐪熷疄鍘熷洜鍙兘鍦ㄥ叾浠栧湴鏂?
- 搴旇鍦?DevUI 鐨?`MessageMapper` 灞傞潰娣诲姞閿欒澶勭悊
- 涓嶆槸鎵€鏈夐棶棰橀兘鑳介€氳繃淇敼鏁版嵁缁撴瀯瑙ｅ喅

