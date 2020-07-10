![](https://raw.githubusercontent.com/createskyblue/NumRandomCube/master/%E7%A4%BA%E6%84%8F%E5%9B%BE/%E5%B0%81%E9%9D%A2.JPG)
# 引脚定义图
![](https://raw.githubusercontent.com/createskyblue/NumRandomCube/master/%E7%A4%BA%E6%84%8F%E5%9B%BE/%E4%BE%A7%E8%A7%86%E5%9B%BE.JPG)
# 说明
亮点1 多阶灰度模拟，用最少的资源做出灰阶光立方
亮点2 可以使用串口通讯通过光立方的密度、亮度、闪烁频率直观地显示出CPU占用率（可惜视频拍不出这个效果，想感受得自己做一个）

通过合理调整扫描间隔实现灰度显示，定时器不足采用软件模拟所以存在一定的缺陷：

缺陷1 更精细的灰阶需要牺牲刷新率换取
缺陷2 使用了所有可以用的io驱动，只可以周期性地进行串口通讯交换数据
# 驱动程序
```cpp
/*
  +创天蓝编写 1281702594@qq.com  采用CC协议
  +关于串口通讯
  -当需要时，光立方驱动程序会向上位机发送指令“get”，当上位机收到get后可开始传送cpu使用率(0-100)
  -默认2000ms光立方发送一次刷新请求
  -串口通讯超时默认为1500ms
  -请求状态下向光立方发送串口指令"120"会软重启光立方
  +倘若通讯超时 将清空显示，如果希望通讯超时后显示随机内容请把SerialReadCMD()函数中最后一处出现的"p=0;"改为"p=random(0,100);"即可
  +支持更改光立方大小 默认大小4x4x4 把4改为你想要的即可
  -如果光立方分辨率高于4x4x4需要修改LightCube()函数，因为arduino某些型号并不能直接驱动高于4x4x4的光立方
  -不过不用担心，修改方式很简单，把IO控制的两行代码改成控制所需要的控制寄存器的代码就好
*/
int cube[4][4][4];
int layer[4] = {A5, A4, A3, A2}; //光立方的引脚
int column[4][4] = {
  A1, A0, 13, 12,
  8,  9,  10, 11,
  4,  5,  6,  7,
  0,  1,  2,  3
};

byte LightCycle = 0;
int p = 0;
byte pw;
unsigned long DisplayerTime = 0;
unsigned long RandomCoolTime = 0;
unsigned long SerialReadCMDTimeOut = 0;
void (*resetFunc)(void) = 0;

void BootLogo() {
  cube[3][3][3]=255;

  }

void setup() {
  randomSeed(analogRead(A0));
  SetupCubePin();
  CleanCube();
  BootLogo();
  LightCube();
  //Serial.begin(115200);
}

void loop() {
  DisplayerTime = millis();
  //                                  ↓ 默认的请求刷新时间 单位ms
  while (millis() < DisplayerTime + 1000) {
    LightCube();
    //选择闪烁模式
    //byte RamTimePlusP = 100 - p + random(25, 75);  //CPU使用率低时闪得慢，使用率高时闪得快
    byte RamTimePlusP = random(25, 85 + p); //CPU使用率高时闪得慢，使用率低时闪得快
    while (millis() > RandomCoolTime + RamTimePlusP) {
      RandomCube();
      //PrintCube();
      RandomCoolTime = millis();
    }
  }
  //CleanCube();
  LightCube();
  p = 1000.0/(5.3*random(1, 100)+10);
  //SerialReadCMD();
}
void SetupCubePin() {
  //Serial.println("SetupCubePin");
  for (byte z = 0; z < 4; z++) pinMode(layer[z], OUTPUT);
  for (byte y = 0; y < 4; y++)
    for (byte x = 0; x < 4; x++) pinMode(column[x][y], OUTPUT);
}
void CleanCube() {
  //Serial.println("CleanCube");
  for (byte z = 0; z < 4; z++)
    for (byte y = 0; y < 4; y++)
      for (byte x = 0; x < 4; x++) cube[x][y][z] = 0;
}
void PrintCube() {
  Serial.println("PrintCube");
  for (byte z = 0; z < 4; z++) {
    for (byte y = 0; y < 4; y++) {
      //Serial.println();
      for (byte x = 0; x < 4; x++) Serial.print(cube[x][y][z] + String(" "));
    }
    Serial.println();
  }
}
void RandomCube() {
  //Serial.println("RandomCube");
  //CleanCube();
  int NewLight;
  pw = 0;
  for (byte t = 0; t < 2; t++)
    for (byte z = 0; z < 4; z++)
      for (byte y = 0; y < 4; y++)
        for (byte x = 0; x < 4; x++) {
          if (cube[x][y][z] > 0) cube[x][y][z] -= 10;
          if (cube[x][y][z] < 0) cube[x][y][z] = 0;
          if (random(0, 100) < abs(p - 100 * t)) {
            NewLight = !t * random(15, 100) + cube[x][y][z];
            if (NewLight < 256) cube[x][y][z] = NewLight; else NewLight = 255;
          };
          pw++;
        }
  //Serial.println(float(pw) / 64);
}
void LightCube() {
  //Serial.println("LightCube");
  for (byte z = 0; z < 4; z++) {
    digitalWrite(layer[z], 0);
    for (byte y = 0; y < 4; y++)
      for (byte x = 0; x < 4; x++) {
        if (LightCycle < cube[x][y][z]) digitalWrite(column[x][y], 1); else digitalWrite(column[x][y], 0);
      }
    digitalWrite(layer[z], 1);
  }
  LightCycle += 25;
}
void SerialReadCMD() {
  pinMode(13, OUTPUT);
  digitalWrite(13, 1);
  Serial.begin(115200);
  String inString = "";
  SerialReadCMDTimeOut = millis();
  delay(4);
  while (1) {
    //Serial.println("Get");
    while (Serial.available() > 0) {
      char inChar = Serial.read();
      inString += (char)inChar;
    }
    if (inString != "" || millis() > SerialReadCMDTimeOut + 1500) break;  //收到数据或者连接超时

  }
  //p = 0; //通讯超时后清空光立方
  p = random(1, 10); //通讯超时后开启随机展示模式
  if (inString != "") p = inString.toInt();
  if (p == 110) resetFunc();
  Serial.println(p);
  Serial.end();  //UNO IO不够通讯结束后防止显示异常必须关闭串口，如果你的串口IO与显示IO不是混用可以注释这一项
}

```
