## 举例
**ui.theme包**，修改```Theme.kt```，如下
```
...
import androidx.compose.runtime.Stable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.setValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.ui.graphics.Color

...

private val CustomDarkColor = CustomColors(
    background = DarkThemeBg
)

@Stable
class CustomColors(
    background: Color
){
    var background: Color by mutableStateOf(background)
        private set
}

@Stable
object CustomBackgroundTheme{
    val color: CustomColors
        @Composable
        get() = CustomDarkColor
}
...

```
其中DarkThemeBg是颜色，在**ui.theme**包的```Color.kt```中声明，如下
```
import androidx.compose.ui.graphics.Color

...
val LightThemeBg = Color(0xFFFFFFFF)
val DarkThemeBg = Color(0xFF00FF00)
```
使用，如下
```kotlin
@Composable
fun PersonalInfo(){
    Row(Modifier.background(color = CustomBackgroundTheme.color.background)) {
        ...
    }
}
```

