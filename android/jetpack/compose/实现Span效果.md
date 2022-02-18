## 使用`visualTransformation`属性
```kotlin
class ColorsTransformation() : VisualTransformation {
    override fun filter(text: AnnotatedString): TransformedText {
        return TransformedText(
            buildAnnotatedStringWithColors(text.toString()), 
            OffsetMapping.Identity)
    }
    
    fun buildAnnotatedStringWithColors(text:String): AnnotatedString{
        val words: List<String> = text.split("\\s+".toRegex())// splits by whitespace
        val colors = listOf(Color.Red,Color.Black,Color.Yellow,Color.Blue)
        var count = 0

        val builder = AnnotatedString.Builder()
        for (word in words) {
            builder.withStyle(style = SpanStyle(color = colors[count%4])) {
                append("$word ")
            }
            count ++
        }
        return builder.toAnnotatedString()
    }
}

TextField(
    value = text,
    onValueChange = { text = it },
    visualTransformation = ColorsTransformation()
)
```

