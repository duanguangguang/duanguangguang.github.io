---
title: JAVA基础(十一)Swing
date: 2017-08-21 21:43:10
categories: 
 - JAVA
 - JavaSE
tags:
 - JavaSE
---

## java基础之Swing

J作为Swing组件的前缀，一些重要的组件：

| JFrame     | 窗体组件    |
| ---------- | ------- |
| JPanel     | 面板组件    |
| JLabel     | 标签组件    |
| JTextField | 单行输入框组件 |
| JButton    | 按钮组件    |

<!--more -->

### 一、窗体

Swing的窗体模型：Swing窗体中包含一个主容器，主容器中可以包含其他组件与子容器，子容器还可以有自己的组件与容器

~~~java
public class TestFrame extends JFrame{
	public TestFrame(){
		this.setBounds(50, 50, 50, 100);//设置窗口大小
		this.setTitle("窗口标题");
		this.setBackground(Color.BLACK);//窗口背景色
		this.setAlwaysOnTop(true);//总是在最前面
		this.setResizable(false);//不可调整大小
		this.setDefaultCloseOperation(EXIT_ON_CLOSE);//点击关闭后退出程序
     	//注:如果不指定窗体的默认关闭行为是隐藏窗体而不是退出程序
		this.setVisible(true);//可见
	}
	public static void main(String[] args) {
		new TestFrame();
	}
}
~~~

### 二、布局管理器

在进行布局嵌套时，一般选择JPanel作为子容器

#### 1. 手工布局（null）

null布局组件会一个像素不差的放在指定位置

~~~java
Container c = this.getContentPane();//获取主容器
c.setLayout(null);//布局设为null
~~~

#### 2. BorderLayout边框布局

边框布局最多容纳5个组件，放置在上下左右中间方位，在将组件放入到容器时需要指定方位。JFrame主容器默认使用边框布局

~~~java
Container c = this.getContentPane();//获取主容器
c.add(new JButton("A"),BorderLayout.NORTH);
c.add(new JButton("B"),BorderLayout.SOUTH);
c.add(new JButton("C"),BorderLayout.WEST);
c.add(new JButton("D"),BorderLayout.EAST);
c.add(new JButton("E"),BorderLayout.CENTER);
~~~

#### 3. FlowLayout流式布局

流式布局会将组件按从左到右，从上到下排列，宽度不够会进行折行处理，可以在初始化布局管理器时选择左对齐、右对齐或居中对齐

~~~java
Container c = this.getContentPane();//获取主容器
c.setLayout(new FlowLayout(FlowLayout.CENTER));
c.add(new JButton("A"));
c.add(new JButton("B"));
~~~

#### 4. GridLayout网格布局

网格布局会将组件按行列排列，初始化时需要指定行数列数，并可以选择行间距与列间距

~~~java
Container c = this.getContentPane();//获取主容器
c.setLayout(new GridLayout(3,2,20,20));//3行2列，行列间距都为20
c.add(new JButton("A"));
c.add(new JButton("B"));
~~~

### 三、事件处理

事件监听接口：

| ActionListener | 通过鼠标或键盘选择按钮                     |
| -------------- | ------------------------------- |
| MouseListener  | 监听鼠标按下、松开、移入、移出等事件              |
| KeyListener    | 监听键盘按下、松开等事件                    |
| FocusListener  | 监听组件获取、失去焦点事件                   |
| WindowListener | 监听窗体最大化、最小化、打开、关闭等事件，仅对JFrame有效 |

~~~java
public class ClickAndExit extends JFrame{
  JButton btnClose = new JButton("关闭");
  //创建ActionListener接口的匿名实现类
  ActionListener listener = new  ActionListener(){
    public void actionPerformed(ActionEvent e){
      System.exit(0);
    }
  };
  
  public ClickAndExit(){
    this.setBounds(50,50,250,250);
    Container c = this.getContentPane();//获取主容器
    c.add(btnClose);
    //将监听加入按钮的监听器列表
	btnClose.addActionListener(listener);
    //btnClose.addActionListener(new ActionListener(){
    //还可以通过创建匿名对象的方式});
    this.setDefaultCloseOperation(EXIT_ON_CLOSE);
    this.setVisible(true);
  }
  
  public static void main(String[] args){
  	new ClickAndExit();
  }
}
~~~

### 四、其他组件

~~~java
public class OtherFrame extends JFrame{
  ButtonGroup bg = new ButtonGroup();
  JRadioButton rbMale = new JRadioButton("男");
  JRadioButton rbFemale = new JRadioButton("女");
  
  public OtherFrame(){
  	bg.add(rbMale);
    bg.add(rbFemale);
    this.setBounds(50,50,250,250);
    Container c = this.getContentPane();//获取主容器
    c.setLayout(new GridLayout(5,2,10,10));
    //单选框
    c.add(new JLabel("性别"));
    JPanel panelGender = new JPanel();
    c.add(panelGender);
    panelGender.add(rbMale);
    panelGender.add(rbFemale);
    
    //复选框
    c.add(new JLabel("爱好"));
    JPanel panelFav = new JPanel();
    c.add(panelFav);
    panelFav.add(new JCheckBox("篮球"));
    panelFav.add(new JCheckBox("足球"));
    
    //下拉框
    c.add(new JLabel("学历"));
    c.add(new JComboBox("小学，中学，大学".split(",")));
    
    //数字选择
    c.add(new JLabel("年龄"));
    c.add(new JSpinner());
    
    //数字滑动
    c.add(new JLabel("年龄"));
    c.add(new JSLider(0,50,100));
    
    this.setDefaultCloseOperation(EXIT_ON_CLOSE);
    this.setVisible(true);
  }
}
~~~

### 五、内置对话框

#### 1. JOptionPane

最常用的对话框，可用于弹出消框，输入框和选择框

~~~java
public class Alert extends JFrame{
  public static void main(String[] args){
    //输入框
    String msg = JOptionPane.showInputDiglog("输入的话","默认值");
    //选择框
    int yesno = JOptionPane.showConfirmDialog(null,"你选择的是" + msg +"吗?");
    //消息框
    JOptionPane.showMessageDialog(null,msg);
  }
}
~~~

#### 2. JFileChooser

JFileChooser可以选择磁盘上的文件，能够进行单选或多选，并返回结果

~~~java
public class FileCh extends JFrame{
  public static void main(String[] args){
    JFileChooser jfc = new JFileChooser();
    int result = jfc.showOpenDiaLog(null);
    if(result != JFileChooser.CANCEL_OPTION){
      File f = jfc.getSelectedFile();
      System.out.println("你选择的文件是：" +f.getAbsolutePath());
    }else{
      System.out.println("你取消了选择");
    }
  }
}
~~~

