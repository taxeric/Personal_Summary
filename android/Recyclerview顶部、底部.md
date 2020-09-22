假设接口数据一共有10条（不包括顶边和底边），项目需要显示top和foot，可以这么写：
1. 在加载网络数据前先add一条空数据
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
    private val mutableList: MutableList<String>
): RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    private var listener: RvOnItemClickListener ?= null

    fun setItemClickListener(listener: RvOnItemClickListener){
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
            holder.itemView.setOnClickListener{
                if (listener != null){
                    holder.itemView.click_load_more.text = "加载中..."
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
}
```

activity简要代码如下
```java
class HomeFragment: BaseFragment(), RvOnItemClickListener {

    private val mutableList = mutableListOf<String>()
    private lateinit var adapter: HomeRvAdapter

    override fun setLayout(): Int = R.layout.fragment_home

    override fun initData() {
        home_rv.layoutManager = LinearLayoutManager(context)
        mutableList.add("顶部不显示")
        create()
        adapter = HomeRvAdapter(context!!, mutableList)
        adapter.setItemClickListener(this)
        home_rv.adapter = adapter
    }

    private fun create(){
        for (i in 1..10){
            mutableList.add("$i")
        }
    }

    override fun onItemClick(position: Int) {
        LogUtils.i("position = $position")
        create()
        adapter.notifyDataSetChanged()
    }
}
```
