```go
func main() {  
  mp := getg().m  
  
  // Racectx of m0->g0 используется только как родительский контекст для главной горутины.
  // Он не должен использоваться ни для чего другого.
  mp.g0.racectx = 0  
  
  // Максимальный размер стека 1 GB on 64-bit, 250 MB на 32-битной системе.  
  // Используются десятичные числа вместо двоичных, потому что   
  // они выглядят более аккуратно в сообщении об ошибке переполнения стека
	if goarch.PtrSize == 8 {  
	   maxstacksize = 1000000000  
	  } else {  
	   maxstacksize = 250000000  
	  }  
  
  // Верхний предел максимального размера стека.
  // Используется для предотвращения случайных сбоев 
  // после вызова SetMaxStack и попытки аллоцировать слишком большой стек,
  // поскольку stackalloc работает с 32-битными размерами.  
  maxstackceiling = 2 * maxstacksize  
  
  // Разрешение на запуск новых машин (M) - функцией newproc
  mainStarted = true  
  
  if haveSysmon {  
   systemstack(func() {  
    newm(sysmon, nil, -1)  
   }) 
  }  
  
  // Закрепляет main горутину за основным основным потоком ОС 
  // во время инициализации. Большинству программ это не важно, но некоторым необходимо, 
  // чтобы определённые вызовы выполнялись именно в главном потоке.  
  // Такие программы могут обеспечить выполнение main.main в главном потоке,
  // вызвав runtime.LockOSThread во время инициализации,
  // чтобы сохранить эту привязку.
    
  lockOSThread()  
  
  if mp != &m0 {  
   throw("runtime.main not on m0")  
  }  
  
  // Фиксация момента запуска мира (программы).
  // Должна быть выполнена до вызова doInit,
  // чтобы инициализация (init) могла быть корректно отслежена 
  // в трассировке.
  
  runtimeInitTime = nanotime()  
  if runtimeInitTime == 0 {  
   throw("nanotime returning zero")  
  }  
  
  if debug.inittrace != 0 {  
   inittrace.id = getg().goid  
   inittrace.active = true  
  }  
  
  doInit(runtime_inittasks) // Должен выполниться перед defer.  
  
  // Отложить разблокировку, чтобы при выполнении runtime.Goexit
  // во время init разблокировка тоже прошла.
  
  needUnlock := true  
  defer func() {  
   if needUnlock {  
    unlockOSThread()  
   }  
  }()  
  
  gcenable()  
  
  main_init_done = make(chan bool)  
  if iscgo {  
   if _cgo_pthread_key_created == nil {  
    throw("_cgo_pthread_key_created missing")  
   }  
  
   if _cgo_thread_start == nil {  
    throw("_cgo_thread_start missing")  
   }  
   if GOOS != "windows" {  
    if _cgo_setenv == nil {  
     throw("_cgo_setenv missing")  
    }  
    if _cgo_unsetenv == nil {  
     throw("_cgo_unsetenv missing")  
    }  
   }  
   if _cgo_notify_runtime_init_done == nil {  
    throw("_cgo_notify_runtime_init_done missing")  
   }  
  
   // Устанавливает указатель на функцию C x_crosscall2_ptr, чтобы он указывал на crosscall2. 
   if set_crosscall2 == nil {  
    throw("set_crosscall2 missing")  
   }  
   set_crosscall2()  
  
   // Запускает «шаблонный» поток на случай, если мы входим в Go из
   // потока, созданного на стороне C, и требуется создать новый поток.   
   startTemplateThread()  
   cgocall(_cgo_notify_runtime_init_done, nil)  
  }  
  
  // Выполнить задачи инициализации. В зависимости от режима сборки этот
  // список может формироваться разными способами, но он всегда   
  // содержит задачи инициализации, рассчитанные линкером для всех 
  // пакетов программы (за исключением тех, что добавляются во время выполнения 
  // при помощи пакета plugin). Проход по модулям выполняется в порядке зависимостей - 
  // в том же порядке, в каком они инициализируются динамическим 
  // загрузчиком, то есть в порядке добавления в связный список moduledate. 
  for m := &firstmoduledata; m != nil; m = m.next {  
   doInit(m.inittasks)  
  }  
  
  // Отключить трассировку инициализации после завершения инициализации main
  // чтобы избежать издержек, связанных со сбором статистики в malloc и newproc.
  inittrace.active = false  
  
  close(main_init_done)  
  
  needUnlock = false  
  unlockOSThread()  
  
  if isarchive || islibrary {  
   // Программа, скомпилированная с флагом -buildmode=c-archive или c-shared  
   // содержит функцию main, но она не выполняется.
   if GOARCH == "wasm" {  
    // На платформе Wasm (WebAssembly) вызов pause заставляет выполнение вернуться к хосту.  
    
    // В отличие от обратных вызовов (callbacks) в cgo, где новые потоки (M) создаются по мере необходимости,  
    // в Wasm существует только один поток исполнения (M). 
    
    // Поэтому мы сохраняем этот поток (M) 
    // и текущую горутину (G) для обработки обратных вызовов из хоста.
    
    // Использование **указателя стека (SP)** вызывающей функции 
    // позволяет «развернуть» (unwind) текущий фрейм    
    // и вернуться к функции goexit. 
    
    // Смещение на 16 байт(-16) объясняется так:
    // 8 байт занимают фиктивный адрес возврата (fake return PC) функции goexit;  
    // ещё 8 байт снимаются (pop) в эпилоге функции pause.
    pause(sys.GetCallerSP() - 16) // не должен возвращаться 
    panic("unreachable")  
   }  
   return  
  }  
  fn := main_main // Выполняем косвенный вызов (indirect call), 
				  // поскольку компоновщику (linker) неизвестен адрес пакета main
				  // во время размещения (развёртывания) кода рантайма (runtime).
  fn()  
  if raceenabled {  
   runExitHooks(0) // Выполняем hook-функции (хуки) сейчас, 
				   // тк racefini не возвращает управление (не делает возврата).
   racefini()  
  }  
  
  // Сделать так, чтобы программа с потенциальными гонками (racy client program) 
  // работала корректно: если паника происходит  
  // в другой горутине одновременно с возвратом из main,
  // нужно позволить другой горутине завершить вывод трассировки паники.
   
  // После этого она сама завершит выполнение.
  if runningPanicDefers.Load() != 0 {  
   // Выполнение отложенных функций (deferred functions) не должно занимать много времени.
   for c := 0; c < 1000; c++ {  
    if runningPanicDefers.Load() == 0 {  
     break  
    }  
    Gosched()  
   }  
  }  
  if panicking.Load() != 0 {  
   gopark(nil, nil, waitReasonPanicWait, traceBlockForever, 1)  
  }  
  runExitHooks(0)  
  
  exit(0)  
  for {  
   var x *int32  
   *x = 0  
  }  
}
```
