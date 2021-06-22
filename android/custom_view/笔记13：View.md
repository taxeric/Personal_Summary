当disptchTouchEvent(MotionEvent event)被调用时，说明事件已经被分发到View中；  
当onTouchEvent(MotinoEvent event)被调用时，表示用户点击/按下等事件

被分发到view后，会
1. 判断是否有可响应焦点，如果没有直接返回false，事件未被消费
2. 若有可响应焦点，则验证触摸事件是否符合安全策略，不符合，如上
3. 符合策略，判断是否为鼠标事件，是，返回true，表示被消费
4. 不是鼠标事件，判断是否注册TouchListener，监听onTouch返回值


