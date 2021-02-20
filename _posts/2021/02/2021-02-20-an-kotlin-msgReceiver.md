---
title: (안드로이드/코틀린) Service와 Fragment/Activity 사이에서 데이터 전달 및 통신방법
tags: [Android, Kotlin]
style: fill
color: dark
description: 안드로이드 코틀린에서 Service와 Fragment/Activity 사이에서 데이터 전달 및 통신방법
---

## Activity/Fragment to Service 데이터 전달 및 통신방법
서비스와 액티비티/프래그먼트 사이에서 데이터를 전달하기 위해서는 리시버가 필요합니다. 두 컴포넌트 사이에 사용할 인텐트 필터를 정하고 이를 통해 데이터를 주고 받으면서 통신할 수 있습니다.


## 실습하기
**1. 서비스 코드 작성하기**
- 모바일 뿐만 아니라 다른 파트의 많은 개발자들이 Utils 클래스를 만들고 그 안에 자주 쓰이는 코드들을 넣고 다니며 프로젝트 개발 속도를 늘린다.

```javascript
class MyService : Service() {

    override fun onBind(intent: Intent): IBinder? {
        return null
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {

        Log.e("CJS", "???")

        sendMessage("num1", "HI")
        sendMessage("num2", "BYE")

        return super.onStartCommand(intent, flags, startId)
    }

    fun sendMessage(key: String, msg: String) {
        val intent = Intent("test")
        intent.putExtra("key", key)
        intent.putExtra("msg", msg)
        LocalBroadcastManager.getInstance(this).sendBroadcast(intent)
    }
}
```

- 서비스에서는 Intent에 (key, value) 로 전달할 값을 정하고 인텐트 필터를 "test"로 설정합니다.
- LocalBroadcastManager를 통해서 해당 인텐트를 리시버로 전달합니다.


**2. Activity/Fragment 코드 작성하기**

```javascript
class DashboardFragment : Fragment() {

    private lateinit var dashboardViewModel: DashboardViewModel

    override fun onCreateView(
            inflater: LayoutInflater,
            container: ViewGroup?,
            savedInstanceState: Bundle?
    ): View? {
        dashboardViewModel =
                ViewModelProvider(this).get(DashboardViewModel::class.java)
        val root = inflater.inflate(R.layout.fragment_dashboard, container, false)
        val textView: TextView = root.findViewById(R.id.text_dashboard)
        dashboardViewModel.text.observe(viewLifecycleOwner, Observer {
            textView.text = it
        })
        return root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        //서비스 실행
        val serviceIt = Intent(activity, MyService::class.java)
        activity?.startService(serviceIt)

        //등록
        activity?.let { register(it) }
    }

    private fun register(ctx: Context) {
        LocalBroadcastManager.getInstance(ctx).registerReceiver(
                testReceiver, IntentFilter("test")
        )
    }

    fun unRegister(ctx: Context) {
        LocalBroadcastManager.getInstance(ctx).registerReceiver(
                testReceiver, IntentFilter("test")
        )
    }

    override fun onDestroy() {
        super.onDestroy()
        activity?.let { unRegister(it) }
    }

    private val testReceiver: BroadcastReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {

            val msg = intent.getStringExtra("msg")

            when (intent?.getStringExtra("key")) {

                "num1" -> {
                    Toast.makeText(activity, "Num1 from service $msg", Toast.LENGTH_SHORT).show()
                }
                "num2" -> {
                    Toast.makeText(activity, "Num2 from service $msg", Toast.LENGTH_SHORT).show()
                }
            }
        }
    }
}
```
- 서비스에서 보내는 인텐트를 수신할 testReceiver를 작성하고 key에 따라 분류하고 해당 msg를 출력하면 됨
- LocalBroadcastManager를 통해 testReciver를 등록하고 test로 오는 IntentFilter 수신대기
- 메모리 낭비가 발생할 수 있으므로 프래그먼트/액티비티가 종료 또는 사용이 끝나게 되면 unRegister()를 꼭 해주기!

## 결론
- **등록 & 해제 까먹지 않기**
