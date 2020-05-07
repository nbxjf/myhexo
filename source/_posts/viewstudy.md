---
title: 自定义属性、布局、控件初学
date: 2016-05-28 17:14:15
categories: Android
tags:
	- 自定义属性
	- 自定义控件
---
Android开发中难免遇到需要自定义控件的需求，有些是产品的要求在Android标准控件库中没有满足要求的，有些是开发过程中没有代码的可复用，自己定义的。
<!--more-->

# 自定义属性
* 对于要自定义的属性，如需要自定义TextView的onFocus属性，默认是false，即不获取焦点状态，当我想默认获取焦点
	* 创建工具类，继承自TextView，实现构造函数

			public class FocusTextView  extends TextView{
    		//使用java创建控件
    		public FocusTextView(Context context) {
        		super(context);
   			 }

   			// 由系统调用（上下文环境+属性集）
    		public FocusTextView(Context context, AttributeSet attrs) {
        		super(context, attrs);
    		}

    		//由系统调用（上下文环境+属性集+布局文件中的定义的样式文件构造方法）
    		public FocusTextView(Context context, AttributeSet attrs, int defStyleAttr) {
        		super(context, attrs, defStyleAttr);
   			 }

    * 重写isFocus方法

			//重写获取焦点的方法，返回true，默认获取焦点
   			 @Override
   		 	public boolean isFocused() {
       		 	return true;
   		 		}
			}
	* 将需要使用的空间名换成定义的工具类的包名即可

# 自定义布局
* ![](http://i.imgur.com/R5h6ehV.png)假设我要实现类似的布局，若该布局方式需要重复很多遍，，如实现九宫格效果，写9次是不现实的，就需要去自定义这样的一个布局
	* 将需要自定义的布局抽取出来并保存到一个xml布局文件中

			<ImageView
        		android:id="@+id/iv_icon"
        		android:background="@drawable/home_apps"
        		android:layout_width="wrap_content"
        		android:layout_height="wrap_content" />
    		<TextView
       		 	android:id="@+id/tv_title"
        		android:text="标题"
        		android:textColor="#000"
        		android:layout_width="wrap_content"
        		android:layout_height="wrap_content" />

	* 建立适配器，将布局文件填充到适配器中

			
			class MyAdapter extends BaseAdapter{

        		@Override
       			public int getCount() {
            		return titleStr.length;
        		}

        		@Override
        		public Object getItem(int position) {
            		return titleStr[position];
        		}
	
        		@Override
        		public long getItemId(int position) {
            		return position;
        		}

        		@Override
        		public View getView(int position, View convertView, ViewGroup parent) {
            		View view = View.inflate(getApplicationContext(),R.layout.gridview_item,null);
            		ImageView iv_icon = (ImageView) view.findViewById(R.id.iv_icon);
            		TextView tv_title = (TextView) view.findViewById(R.id.tv_title);
            		tv_title.setText(titleStr[position]);
            		iv_icon.setBackgroundResource(img[position]);
            		return view;
        		}
   			}
		* 通过View.inflate函数将布局文件填充带View对象中
	* 给布局文件添加适配器

			
			gv_home.setAdapter(new MyAdapter());

# 自定义控件
* 自定义控件的强大应该就不需要讲了，我觉得自定义控件应该是用的最多的吧，初学的我还不是很了解。参照源码中的格式，大致了解到自定义控件的格式

		<resources>
    		<declare-styleable name="">
        		<attr name="" format=""></attr>
        		<attr name="" format=""></attr>
        		<attr name="" format=""></attr>
    		</declare-styleable>
		</resources>
* 参照这种格式，declare-styleable对应的是包名吧，attr的name自然就是属性名跟属性格式了，那么这样我就可以自定义一个控件
* 
			<resources>
    			<declare-styleable name="com.declan.mobilesafer.view.SettingItemview">
        			<attr name="destitle" format="string"></attr>
        			<attr name="desoff" format="string"></attr>
        			<attr name="deson" format="string"></attr>
    			</declare-styleable>
			</resources>
* 引用是在xmlns设置时，仿照xmlns:android="http://schemas.android.com/apk/res/android"的格式进行自己命名，这样就应该比好理解了。
	* android="http://schemas.android.com/apk/res/android是安卓源码定义的命名空间

		<com.declan.mobilesafer.view.SettingItemview
        	xmlns:mobilesafer="http://schemas.android.com/apk/res/	com.declan.mobilesafer"
        	mobilesafer:destitle=""
        	mobilesafer:desoff = ""
        	mobilesafer:deson = ""
        	android:layout_width="match_parent"
        	android:layout_height="wrap_content">
    	</com.declan.mobilesafer.view.SettingItemview>
* 获取自定义控件的属性值		
	* 循环遍历

			//通过属性索引获取属性名称&属性值
			for(int i=0;i<attrs.getAttributeCount();i++){
				Log.i(tag, "name = "+attrs.getAttributeName(i));
				Log.i(tag, "value = "+attrs.getAttributeValue(i));
				Log.i(tag, "分割线 ================================= ");
			}
	* 也可以通过属性获取属性名称&名空间
	
			attrs.getAttributeValue(NAMESPACE, "destitle")
			mDesoff = attrs.getAttributeValue(NAMESPACE, "desoff");
			mDeson = attrs.getAttributeValue(NAMESPACE, "deson");

