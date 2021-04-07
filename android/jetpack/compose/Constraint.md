## 约束布局
#### 使用
需要引入
```xml
implementation("androidx.constraintlayout:constraintlayout-compose:1.0.0-alpha03")
```
例子
```java
@Preview
@Composable
fun ConstraintLayoutContent() {
    ConstraintLayout {
        //这里的button和text猜测是类似id的东西
        val (button, text) = createRefs()
        val test = createRef()

        Button(
            onClick = { /* Do something */ },
            modifier = Modifier.constrainAs(button) {
                top.linkTo(parent.top, margin = 16.dp)
            }
        ) {
            Text("Button")
        }

        Text("Text", Modifier.constrainAs(text) {
            top.linkTo(button.bottom, margin = 10.dp)
            start.linkTo(button.end)//起点在button的末尾
        })

        Text(text = "测试文本", Modifier.constrainAs(test) {
            baseline.linkTo(text.baseline)//与text对齐
            end.linkTo(text.start)//末尾在text的起点
        })
    }
}
```
## 两种构造函数
```java
@androidx.compose.runtime.Composable 
public fun ConstraintLayout(
  odifier: androidx.compose.ui.Modifier /* = compiled code */, 
  content: @androidx.compose.runtime.Composable() (androidx.constraintlayout.compose.ConstraintLayoutScope.() -> kotlin.Unit)
): kotlin.Unit { /* compiled code */ }

@androidx.compose.runtime.Composable 
public fun ConstraintLayout(
  constraintSet: androidx.constraintlayout.compose.ConstraintSet, 
  modifier: androidx.compose.ui.Modifier /* = compiled code */, 
  content: @androidx.compose.runtime.Composable() () -> kotlin.Unit
): kotlin.Unit { /* compiled code */ }
```

> 到目前为止我们并没有见到过Compose类似xml中那种给控件命名id的形式，那既然Compose是约束布局，必须要有个控件的编号或者ID，才能实现各种约束条件， 目前在ConstraintLayout中有两种方式来创建这种编号的方式：
> 
> - 如果使用的参数是ConstraintLayoutScope()的方式，那么可以使用如下函数创建编号并应用给控件：  
> Modifier.constrainAs()和createRef()、createRefs()；
> - 如果使用的参数是ConstraintSet的方式，那么则可以使用如下方式给控件创建ID和然后根据ID创建编号：  
> Modifier.layoutId()和createRefFor()；
### 使用
```java
@Composable
fun ConstraintLayoutDemo() {
    ConstraintLayout(
        modifier = Modifier.fillMaxSize()
    ) {

        val guideline = createGuidelineFromStart(0.2f)
        val (box1, box2) = createRefs()

        Box(
            modifier = Modifier.fillMaxSize()
                .background(color = Color.Yellow)
                .constrainAs(box1) {
                    end.linkTo(guideline)
                }
        )

        Box(
            modifier = Modifier.fillMaxSize()
                .background(color = Color.Red)
                .constrainAs(box2) {
                    start.linkTo(guideline)
                }
        )
}
```
```java
@Composable
fun ConstraintLayoutIdDemo() {
    ConstraintLayout(
        ConstraintSet {
            val box1 = createRefFor("box1")
            val box2 = createRefFor("box2")
            val box3 = createRefFor("box3")

            constrain(box1) {
                top.linkTo(parent.top)
                start.linkTo(parent.start)
            }

            constrain(box2) {
                top.linkTo(box1.bottom)
                start.linkTo(parent.start)
            }

            //屏障，对齐用的，哪个宽用哪个
            val barrier = createEndBarrier(box1, box2)

            constrain(box3) {
                start.linkTo(barrier)
                top.linkTo(box1.top)
                bottom.linkTo(box2.bottom)
            }
        }
    ) {
        Box(
            modifier = Modifier.layoutId("box1")
                .background(color = Color.Red)
                .width(100.dp)
                .height(100.dp)
        )
        Box(
            modifier = Modifier.layoutId("box2")
                .background(color = Color.Yellow)
                .width(150.dp)
                .height(100.dp)
        )
        Box(
            modifier = Modifier.layoutId("box3")
                .background(color = Color.Blue)
                .width(200.dp)
                .height(100.dp)
        )
    }
}
```
