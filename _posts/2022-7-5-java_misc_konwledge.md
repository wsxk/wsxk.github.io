---
layput: post
tags: [re]
title: "java misc knowledge"
date: 2022-7-5
author: wsxk
comments: true
---

记录一下java逆向时学到的一些东西。<br>

- [jdk 源码<br>](#jdk-源码)
- [@override<br>](#override)
- [AWT和Swing<br>](#awt和swing)
  - [控件绑定<br>](#控件绑定)
- [reference<br>](#reference)

# jdk 源码<br>
记录一下jdk源码的位置<br>
其实在你安装java环境时，会自动安装一份源码（以压缩包形式存放于你的java目录里）<br>
在目录里搜索src.zip<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2022-6-27-DNS/20220705203625.png)
解压后就可以查看响应的jdk源码（但是真的是又臭又长，解压开来有212MB，看都能给你看麻）

# @override<br>
字面意思，表示该方法可以被重载

# AWT和Swing<br>
AWT和Swing都是java中的package<br>
> AWT(Abstract Window Toolkit)：抽象窗口工具包，早期编写图形界面应用程序的包。
> Swing ：为解决 AWT 存在的问题而新开发的图形界面包。Swing是对AWT的改良和扩展

java的gui应用程序会使用如上两个包来进行开发<br>
## 控件绑定<br>
可以使用下面的demo感受一下控件的绑定和监听<br>
```java
import java.awt.FlowLayout;
import java.awt.event.ActionEvent;
import java.awt.event.ActionListener;
import java.awt.event.MouseAdapter;
import java.awt.event.MouseEvent;
import java.awt.event.MouseListener;

import javax.swing.JButton;
import javax.swing.JFrame;

public class Test {
	public static void main(String[] args) {
		new MyFrame();
	}
}

class MyFrame extends JFrame implements ActionListener, MouseListener {
	public MyFrame() {
		this.setTitle("no button click");
		this.setVisible(true);
		this.setLocationRelativeTo(null);
		this.setLayout(new FlowLayout());
		this.setSize(500, 200);

		JButton button1 = new JButton("mouse click");
		button1.addMouseListener(this);

		JButton button2 = new JButton("action listen");
		button2.addActionListener(this);

		JButton button3 = new JButton("mouse fit");
		button3.addMouseListener(new MouseAdapter() {
			public void mousePressed(MouseEvent e) {
				MyFrame.this.setTitle("mouse fit answer");
			}
		});

		this.add(button1);
		this.add(button2);
		this.add(button3);
	}

	@Override
	public void mouseClicked(MouseEvent e) {
	}

	@Override
	public void mouseEntered(MouseEvent e) {
	}

	@Override
	public void mouseExited(MouseEvent e) {
	}

	@Override
	public void mousePressed(MouseEvent e) {
		this.setTitle("mouse click answer");
	}

	@Override
	public void mouseReleased(MouseEvent e) {
	}

	@Override
	public void actionPerformed(ActionEvent e) {
		this.setTitle("action listen answer");
	}
}
```
# reference<br>
[https://blog.csdn.net/qq_32725491/article/details/78701620](https://blog.csdn.net/qq_32725491/article/details/78701620)
<br>
[https://blog.csdn.net/new_Aiden/article/details/50674045](https://blog.csdn.net/new_Aiden/article/details/50674045)