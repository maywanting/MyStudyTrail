title: "用消息传输实现进程间通信"
date: 2015-11-16 21:53:49
tags:
- C#
categories:
- study
---

前段时间操作系统的实验课要求我们实现windows下进程间的通信，正好趁这个温习以下C#的写法，啊哈哈~····<br/><br/>简单的说就是程序p1负责获取用户输入的要求，是打开程序并输入程序名或是关闭程序并输入程序名。然后p1将获取的数据通过进程间通信传给进程p2，p2负责程序的调度。

> ##p1进程

p1要干的事情就是收集用户输入的信息，然后在发送给p2，这有点像web里面的客户端也就是前台哈。

###1、收集前台信息包装

窗体设计如下，就不贴代码了，拖拖控件还是非常简单的
![image](../../../../img/20151115_2.png)

这里获取数据和asp.net一样，为了方便传输，将打开或关闭用数字表示，然后程序名string表示

```java
int flag = (textBox1.Text.ToString().Equals("open")) ? 1 : 0;
String name = textBox2.Text.ToString();

```

进程间通信有三种方式，一种是共享内存，还有消息传输，还有剪贴板，这里面采用消息传输，也就是说用sendMessage这个函数，首先得调用动态链接库里的函数。

```java
//要传输数据，必须得用这样的结构体！
public struct My_lParam
{
    public IntPtr dwData;
    public int cbData; //open or close
    [MarshalAs(UnmanagedType.LPStr)]
    public string lpData;
}

[DllImport("user32.dll", EntryPoint = "SendMessage")]
private static extern int SendMessage(IntPtr wnd, int msg, int wP, ref My_lParam lP);

```

###2、发送数据

然后就是传输数据，这里是通过button点击事件触发传送，如何找到p2进程则是通过获取当前所有进程的信息，然后一一排查它的名字然后找到p2

```java
private void button1_Click(object sender, EventArgs e)
{
    Process[] pros = Process.GetProcesses(); //获取本机所有进程
    for (int i = 0; i < pros.Length; i++)
    {
        if (pros[i].ProcessName == "p2") //名称为ProcessCommunication的进程
        {
            //sendmessage
        }
    } 
}

```

###3、p1完整代码

```java
public struct My_lParam
{
    public IntPtr dwData;
    public int cbData; //open or close
    [MarshalAs(UnmanagedType.LPStr)]
    public string lpData;
}

[DllImport("user32.dll", EntryPoint = "SendMessage")]
private static extern int SendMessage(IntPtr wnd, int msg, int wP, ref My_lParam lP);

private void button1_Click(object sender, EventArgs e)
{
	//数据包装
    My_lParam lp = new My_lParam();
    int flag = (textBox1.Text.ToString().Equals("open")) ? 1 : 0;
    lp.dwData = (IntPtr)100;
    lp.lpData = textBox2.Text.ToString();
    byte[] sarr = System.Text.Encoding.UTF8.GetBytes(lp.lpData);
    int len = sarr.Length;
    lp.cbData = len + 1;

    Process[] pros = Process.GetProcesses(); //获取本机所有进程
    for (int i = 0; i < pros.Length; i++)
    {
        if (pros[i].ProcessName == "p2") //名称为ProcessCommunication的进程
        {
            IntPtr hWnd = pros[i].MainWindowHandle; //获取ProcessCommunication.exe主窗口句柄
            SendMessage(hWnd, 0x4A, flag, ref lp); //点击该按钮，以文本框数据为参数，向Form1发送WM_KEYDOWN消息
        }
    } 
}

```
注意:
- sendMessage的第二个参数必须得是0x4A，因为0x4A才表示向另一个进程传输数据
- 因为用了0x4A，所以sendMessage的第四个参数必须得是My_lParam这个结构体的样子

详细看[官方文档](https://msdn.microsoft.com/en-us/library/ms649011.aspx)

> ##p2进程

p2进程就是接收来自p1发送的数据，然后根据发送的指令，如果收到第三个参数是1，则创建一个进程，否则结束该进程 <br/>

p2界面

![image](../../../../img/20151115_1.png)

```java
public struct My_lParam
{
    public IntPtr dwData;
    public int cbData; //open or close
    [MarshalAs(UnmanagedType.LPStr)]
    public string lpData;
}

//不断捕捉信号
protected override void DefWndProc(ref Message m)
{
    base.DefWndProc(ref m);
    if (m.Msg == 0x4A)
    {
        My_lParam mystr = new My_lParam();
        Type mytype = mystr.GetType();
        mystr = (My_lParam)m.GetLParam(mytype);
        string threadName = mystr.lpData;  //获取数据
        int flag = m.WParam.ToInt32();

        if (flag == 1)
        {
            Process p = new Process();

            //进程默认是放在同一根目录下。
            p.StartInfo.FileName = System.IO.Path.Combine(Application.StartupPath, threadName + ".exe");  
            p.StartInfo.Arguments = threadName + ".exe";
            p.StartInfo.UseShellExecute = true;
            p.Start();
        }
        else
        {
            Process[] p = Process.GetProcessesByName(threadName);
            for (int i = 0; i < p.Length; i++)
            {
                p[i].Kill();
                p[i].Close();
            }
        }
    }

}
```

> ## 碎碎念

然后两个进程同时运行一下,就能实现进程间的通信了，而且还是实时的哦～<br/>
代码写成这样基本就可以交差了，及格妥妥的！


The End~
