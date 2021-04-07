新建```aidl```文件
```java
interface IMyAidlInterface {

    void funA(String str);
    int funB(String str);
}
```
sync，会生成对应的代理类、存根类，路径在app-build-generated-aidl_source_output_dir下  
写Service端
```java
class AIDLService: Service() {

    private var myAIDLStub = MyAIDLStub()

    override fun onCreate() {
        super.onCreate()
        L.i("service create")
    }

    override fun onBind(intent: Intent?): IBinder = myAIDLStub

    override fun unbindService(conn: ServiceConnection) {
        super.unbindService(conn)
        L.i("unbind service")
    }

    override fun onDestroy() {
        super.onDestroy()
        L.i("service destroy")
    }

    /**
     * Binder
     */
    class MyAIDLStub: IMyAidlInterface.Stub(){

        private var value = 1

        override fun funA(str: String) {
            L.i("$str, current value is $value")
        }

        override fun funB(str: String): Int = ++ value
    }
}
```
写ServiceConnectijon
```java
class AIDLServiceConnection: ServiceConnection {

    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        L.i("service connected")
        val binder = service as AIDLService.MyAIDLStub
        binder.funA("Eric")
        L.i("current value is ${binder.funB("a")}")
    }

    override fun onServiceDisconnected(name: ComponentName?) {
        L.i("service disconnected")
    }
}
```
Client调用
```java
    private val serviceConnection = AIDLServiceConnection()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_ipctest)
        bindService()
    }

    private fun bindService(){
        val intent = Intent(this, AIDLService::class.java)
        bindService(intent, serviceConnection, BIND_AUTO_CREATE)
    }

    override fun onDestroy() {
        super.onDestroy()
        unbindService(serviceConnection)
    }
```
