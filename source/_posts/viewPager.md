---
title: 利用viewPager制作轮播图
date: 2016-07-11 17:48:01
categories: Android
tags:
	- viewPager
---

利用ViewPager这个demo实现的是可以左右循环滑动图片，类似于web焦点图的效果，简单大方，效果使用。
<!--more-->

### Layout布局

		```java
		<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
		    xmlns:tools="http://schemas.android.com/tools"
		    android:layout_width="match_parent"
		    android:layout_height="match_parent"
		    tools:context=".MainActivity" >
		
		    <RelativeLayout
		        android:layout_width="match_parent"
		        android:layout_height="160dp" >
		
		        <android.support.v4.view.ViewPager
		            android:id="@+id/viewpager"
		            android:layout_width="match_parent"
		            android:layout_height="match_parent" />
		
		        <LinearLayout
		            android:layout_width="match_parent"
		            android:layout_height="40dp"
		            android:padding="5dp"
		            android:orientation="vertical"
		            android:layout_alignParentBottom="true"
		            android:gravity="center_horizontal"
		            android:background="#66000000" >
		
		            <TextView
		                android:id="@+id/tv_desc"
		                android:layout_width="wrap_content"
		                android:layout_height="wrap_content"
		                android:textColor="@android:color/white"
		                android:singleLine="true" />
		            
		            <LinearLayout 
		                android:id="@+id/ll_point_container"
		                android:layout_width="wrap_content"
		                android:layout_height="wrap_content"
		                android:layout_marginTop="5dp"
		                android:orientation="horizontal"
		                ></LinearLayout>
		        </LinearLayout>
		    </RelativeLayout>
		
		</RelativeLayout>
		```
### MainActivity

 -  ViewPager在Android-support-v4.jar这个jar包下，所以不要导错jar包
 -  自定义MyAdapter继承自PagerAdapter
 -  监听ViewPager的图片改变OnPageChangeListener
 -  通过记录上一次的图片的位置lastPosition来修改焦点原点的显示状态
 
		```java
		package com.declan.viewpager;
		
		import android.support.v4.view.PagerAdapter;
		import android.support.v4.view.ViewPager;
		import android.support.v7.app.AppCompatActivity;
		import android.os.Bundle;
		import android.view.View;
		import android.view.ViewGroup;
		import android.widget.ImageView;
		import android.widget.LinearLayout;
		import android.widget.TextView;
		
		import java.util.ArrayList;
		
		public class MainActivity extends AppCompatActivity implements ViewPager.OnPageChangeListener {
		
		    private ViewPager viewPager;
		    private TextView tv_desc;
		    private LinearLayout ll_point_container;
		    private int[] resourceId;  //图片资源集合
		    private ArrayList<ImageView> list;  //imageview资源集合
		    private LinearLayout.LayoutParams layoutParams;
		    private String[] contentDesc;  //文字描述资源集合
		    private int lastPosition = 0;
		
		    @Override
		    protected void onCreate(Bundle savedInstanceState) {
		        super.onCreate(savedInstanceState);
		        setContentView(R.layout.activity_main);
		
		        initUI();
		        initData();
		        initAdapter();
		
		    }
		    private void initUI() {
		        viewPager = (ViewPager) findViewById(R.id.viewPager);
		        viewPager.addOnPageChangeListener(this);
		        tv_desc = (TextView) findViewById(R.id.tv_desc);
		        ll_point_container = (LinearLayout) findViewById(R.id.ll_point_container);
		    }
		
		    /**
		     * 初始化数据，包括图片资源、文字资源
		     */
		    private void initData() {
		        //图片资源集合
		        resourceId = new int[]{
		                R.drawable.a,R.drawable.b, R.drawable.c, R.drawable.d, R.drawable.e
		        };
		        //文字集合
		        contentDesc = new String[]{
		                "1111",
		                "2222",
		                "3333",
		                "4444",
		                "5555"
		        };
		        ImageView imageView;
		        View pointView;
		        list = new ArrayList<ImageView>();
		        for(int i = 0;i < resourceId.length;i ++){
		            imageView = new ImageView(this);
		            imageView.setImageResource(resourceId[i]);
		            list.add(imageView);
		
		            //给视图添加小白点
		            pointView = new View(this);
		            pointView.setBackgroundResource(R.drawable.selector_bg);
		            layoutParams = new LinearLayout.LayoutParams(10, 10);
		            if(i != 0){
		                layoutParams.leftMargin = 15;
		            }
		            pointView.setEnabled(false);
		            ll_point_container.addView(pointView,layoutParams);
		        }
		    }
		
		    /**
		     * 初始化适配器
		     * 可以将viewPager近似看成横向的listView
		     */
		    private void initAdapter() {
		        //第一次进入初始化数据
		        ll_point_container.getChildAt(0).setEnabled(true);
		        tv_desc.setText(contentDesc[0]);
		        viewPager.setAdapter(new MyAdapter());
		    }
		
		    /**
		     * 自定义适配器,继承自PagerAdapter
		     */
		     class MyAdapter extends PagerAdapter{
		
		         @Override
		         public int getCount() {
		             return list.size();
		         }
		
		         @Override
		         public boolean isViewFromObject(View view, Object object) {
		             return view == object;
		         }
		
		         @Override
		         public Object instantiateItem(ViewGroup container, int position) {
		             ImageView imageView = list.get(position);
		             container.addView(imageView);
		             return imageView;
		         }
		
		         @Override
		         public void destroyItem(ViewGroup container, int position, Object object) {
		             container.removeView((View)object);
		         }
		     }
		
		    @Override
		    public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
		
		    }
		
		    @Override
		    public void onPageSelected(int position) {
		        //新的条目被选中时调用
		        tv_desc.setText(contentDesc[position]);
		        ll_point_container.getChildAt(position).setEnabled(true);
		        ll_point_container.getChildAt(lastPosition).setEnabled(false);
		        lastPosition = position;
		    }
		
		    @Override
		    public void onPageScrollStateChanged(int state) {
		
		    }
		}

		```