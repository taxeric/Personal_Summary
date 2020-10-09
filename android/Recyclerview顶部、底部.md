假设接口数据一共有10条（不包括顶边和底边），项目需要显示top和foot，可以这么写：
1. 在首次加载数据前先add一条空数据用于显示top
2. adapter的`getItemCount`方法返回集合.size + 1
3. adapter的`getItemViewType`方法按需返回：position=0返回顶边，=size返回底边，否则返回正常接口的数据
4. adapter的`onCreateViewHolder`方法按需返回viewHolder
2. 正常请求接口，添加数据

当然，顶边的数据可能是banner，也可通过这种方式写，因为`onCreateViewHolder`是按`getViewType`的返回值来设置viewHolder的

下面的代码实现的是有top，footer点击加载更多的样式

adapter简要代码如下
```java
class HomeRvAdapter constructor(
    private val context: Context,
    private val mutableList: MutableList<String>,
    private val setFootViewText: SetFootViewText
): RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    private var listener: RvListener.OnItemClickListener ?= null

    fun setItemClickListener(listener: RvListener.OnItemClickListener){
        this.listener = listener
    }

    private val TOP_FLAG = 1
    private val NORMAL_FLAG = 0
    private val FOOT_FLAG = -1

    class TopViewHolder constructor(
        view: View
    ): RecyclerView.ViewHolder(view)

    class NormalViewHolder constructor(
        view: View
    ): RecyclerView.ViewHolder(view)

    class FootHolder constructor(
        view: View
    ): RecyclerView.ViewHolder(view)

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        if (viewType == TOP_FLAG){
            return TopViewHolder(LayoutInflater.from(context).inflate(R.layout.home_banner_item, parent, false))
        } else if (viewType == NORMAL_FLAG){
            return NormalViewHolder(LayoutInflater.from(context).inflate(R.layout.home_noraml_item, parent, false))
        } else {
            return FootHolder(LayoutInflater.from(context).inflate(R.layout.rv_item_footer, parent, false))
        }
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        if (holder is NormalViewHolder){
            if (position != 0 && position != mutableList.size) {
                holder.itemView.test_id.text = mutableList[position]
            }
        } else if (holder is FootHolder){
            holder.itemView.click_load_more.text = setFootViewText.loadComplete()
            holder.itemView.setOnClickListener{
                if (listener != null){
                    holder.itemView.click_load_more.text = setFootViewText.loading()
                    listener!!.onItemClick(position)
                }
            }
        }
    }

    override fun getItemCount(): Int = mutableList.size + 1

    override fun getItemViewType(position: Int): Int {
        if (position == 0){
            return TOP_FLAG
        } else if (position == mutableList.size){
            return FOOT_FLAG
        } else {
            return NORMAL_FLAG
        }
    }

    //提供一个接口给外部，用于设置foot文本
    interface SetFootViewText{
        fun loadComplete(): String
        fun loading(): String
    }
}
```

fragment简要代码如下
```java
class HomeFragment: BaseFragment(), RvListener.OnItemClickListener, HomeRvAdapter.SetFootViewText {

    private val mutableList = mutableListOf<String>()
    private lateinit var adapter: HomeRvAdapter
    private val handler = HomeHandler(this)

    override fun setLayout(): Int = R.layout.fragment_home

    override fun initData() {
        home_rv.layoutManager = LinearLayoutManager(context)
        mutableList.add("顶部不显示")
        create()
        adapter = HomeRvAdapter(context!!, mutableList, this)
        adapter.setItemClickListener(this)
        home_rv.adapter = adapter
        home_rv.addOnScrollListener(object: RecyclerView.OnScrollListener(){
        })
    }

    private fun create(){
        for (i in 1..10){
            mutableList.add("$i")
        }
    }

    class HomeHandler constructor(fragment: HomeFragment) :
        UIHandler<HomeFragment>(fragment) {

        override fun handleMessage(msg: Message) {
            super.handleMessage(msg)
            val fragment = get()
            if (msg.what == Config.SUCCESS){
            } else if (msg.what == Config.FAIL){
            }
        }
    }

    override fun onItemClick(position: Int) {
        LogUtils.i("position = $position")
        handler.postDelayed({
            create()
            adapter.notifyDataSetChanged()
        }, 1000)
    }

    override fun loadComplete(): String = "点击加载更多"

    override fun loading(): String = "加载中..."
}
```
## 问题
实际编写时发现一个问题：假设顶部为banner或者别的与正常显示列表数据不一致的item，当划走再划回来时会发现该item又被重新创建了
