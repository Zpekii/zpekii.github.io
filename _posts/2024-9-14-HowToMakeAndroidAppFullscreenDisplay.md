---
titile: 如何使安卓app适配全面屏手机实现完全全屏显示
author: Zpekii
tags: [android]
categories: [android]
---

# 安卓app实现全面屏显示

## 主要修改类文件`MainActivity.java`

### 重写类方法`onCreate`

#### 适配全面屏显示，消除letterbox黑边

- 在`onCreate`方法下编写以下内容，设置app显示布局模式`layoutInDisplayCutoutMode`,官方详细描述:https://developer.android.com/develop/ui/views/layout/display-cutout?hl=zh-cn#handle

- ```java
  
  import androidx.appcompat.app.AppCompatActivity;
  
  public class MainActivity extends AppCompatActivity{
      
      @override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
  
          getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
  
          if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
              getWindow().getAttributes().layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
          }
  
  	}
  }
  
  ```

#### 隐藏系统UI

- ```java
  @RequiresApi(api = Build.VERSION_CODES.R)
  private void hideSystemUI() {
          // 获取当前窗口的装饰视图
          View decorView = getWindow().getDecorView();
  
          // 获取 WindowInsetsController
          WindowInsetsController insetsController = decorView.getWindowInsetsController();
          if (insetsController != null) {
              // 设置系统 UI 行为: 可以通过滑动调出系统UI，短暂显示系统UI
              insetsController.setSystemBarsBehavior(WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
  
              // 隐藏手势条
              insetsController.hide((WindowInsets.Type.systemGestures()));
  
              // 隐藏系统栏：状态栏、导航栏
              insetsController.hide((WindowInsets.Type.systemBars()));
              
          }
  }
  ```

#### 设置默认翻转显示

- ```java
  setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE);
  ```

#### 完整示例

- ```java
  import androidx.appcompat.app.AppCompatActivity;
  import android.view.View;
  import android.view.WindowManager;
  import android.content.pm.ActivityInfo;
  
  public class MainActivity extends AppCompatActivity{
      
      @override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
  
          getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
  
          if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
              getWindow().getAttributes().layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_ALWAYS;
          }
  	
  		setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE);
  	
      	// 隐藏系统 UI
          if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
              hideSystemUI();
          }
          
      }
  	
       @RequiresApi(api = Build.VERSION_CODES.R)
  	private void hideSystemUI() {
          // 获取当前窗口的装饰视图
          View decorView = getWindow().getDecorView();
  
          // 获取 WindowInsetsController
          WindowInsetsController insetsController = decorView.getWindowInsetsController();
          if (insetsController != null) {
              // 设置系统 UI 行为: 可以通过滑动调出系统UI，短暂显示系统UI
              insetsController.setSystemBarsBehavior(WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
  
              // 隐藏手势条
              insetsController.hide((WindowInsets.Type.systemGestures()));
  
              // 隐藏系统栏：状态栏、导航栏
              insetsController.hide((WindowInsets.Type.systemBars()));
              
          }
      }
  }
  ```

## 参考

**stackoverflow**:[android - Fullscreen App With DisplayCutout - Stack Overflow](https://stackoverflow.com/questions/49190381/fullscreen-app-with-displaycutout)

**安卓文档**:

- [Android Developers:支持刘海屏: Views ](https://developer.android.com/develop/ui/views/layout/display-cutout?hl=zh-cn)
- [Android Developers:Activity](https://developer.android.com/reference/android/app/Activity)

