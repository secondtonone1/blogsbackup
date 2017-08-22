---
title: python学习笔记(七) 类和pygame实现打飞机游戏
date: 2017-08-22 11:33:08
categories: 技术开发
tags: [python]
---
python中`类声明`如下：
``` python
class Student(object):
    def __init__(self, name, score):
        self.name = name
        self.score = score
    def printinfo(self):
        print('name is %s, score is %d'%(self.name, self.score))
```
<!--more-->
Student类有两个成员变量，name和score，类的成员函数第一个参数都为self，用来实现成员变量的赋值，__init__是类的初始化函数，初始化成员变量。

类的使用：

``` python
s1 = Student('niuniu',78)
print(s1.name)
print(s1.score)
s1.printinfo()

s2 = Student('gg',100)
s2.printinfo()
s2.age = 100
print('s2 age is %s'%(s2.age))
```

定义s2对象，并且通过s2.age=100，定义了s2的成员变量age，并且初始化为100

类的成员变量有两种方式定义，一个是在__init__函数中，一个是通过类的对象初始化。

类的权限设置：

``` python
class Student2(object):
    def __init__(self, name, score):
        self.__name = name
        self.__score = score
    def getname(self):
        return self.__name
    def getscore(self):
        return self.__score 

    def printinfo(self):
        print('name is %s, score is %d' %(self.__name, self.__score))
```

`__name通过在变量名前边加上__表示该变量为私有变量`，python没有严格的权限限制，只不过通过重命名将__name变为其他的名字了，这样在外部就访问不到这个变量了。

通过添加getname和getscore函数获取成员变量。

``` python
s3 = Student2('s3',99)
s3.printinfo()
name = s3.getname()
print(name)
s3.__name = 'iloveu'
print(s3.__name)
name = s3.getname()
print(name)
s3.printinfo()
```

虽然通过s3.__name = 'iloveu'赋值后，并没有改变类的私有变量__name的数值，因为类的私有变量__name的名称被改为其他的名字，用户无法知道。所以打印出的名字和s3.__name数值不同。

python类同样支持继承

``` python
class Peaple(object):
    def __init__(self, name):
        self.__name = name
    def job(self):
        pass

class Worker(Peaple):
    def job(self):
        print("worker")

class Student(Peaple):
    def job(self):
        print("student")

s1 = Student('student')
s1.job

w1 = Worker('worker')
w1.job

p = Peaple("abc")
w = Worker("abc")
s = Student("abc")
```

可以通过isinstance判断类对象是否是一个类型的实例
``` python
print(isinstance(p, Peaple))
print(isinstance(w,Peaple))
print(isinstance(s,Peaple))
print(isinstance(s,Worker))
print(isinstance(p,Student))
```

`子类对象是基类类型的实例，而基类对象不一定是子类类型的实例`。比如Student继承于Peaple，学生是人，但是人不一定是学生。

类的属性控制：

``` python
class Designer(Worker):
    def __init__(self,name,age):
        self.__name = name
        self.age = age
    def job(self):
        print("Designer")

designer = Designer('David',18) 
```
获取属性，设置属性，判断是否含有某个属性

``` python
#判断类中是否有某个实例print(hasattr(designer, 'age') )
print(hasattr(designer,'job'))
print(hasattr(designer,'name'))

#设置sex属性，属性值为'female'
setattr(designer, 'sex', 'female')

print(designer.sex)

#获取job属性，返回值为job函数对象
fn = getattr(designer, 'job')
#调用fn函数
fn()
```

`类的公有属性，为所有对象共有`，类似于C++的static成员变量

``` python
#通过self变量或者实例自身可以实现实例属性绑定
#在类中直接定义一个变量，这个属性归类所有，类似于C++的static变量。
class Temple(object):
    staticmember = 1000

temp1 = Temple()
#temp1没有自身属性成员staticmember，
#而Temple类含有共享属性
#下面这种方式打印的是类的共有属性
print(temp1.staticmember)
#为实例temp1绑定其自身的成员staticmember
#并且设置数值为2048
temp1.staticmember = 2048
#打印实例temp1的成员staticmember
print(temp1.staticmember)
#打印类的共享成员
print(Temple.staticmember)
#删除实例的属性staticmember
del temp1.staticmember
#打印出类共享的属性
print(temp1.staticmember)
```

可以为类绑定成员函数，也可以只为类的一个实例绑定成员函数

``` python
def setage(self, age):
    self.__age = age
def getage(self):
    return self.__age
###给实例绑定方法
temp1.setage = MethodType(setage, temp1)
temp1.getage = MethodType(getage,temp1)
temp1.setage(125)
print(temp1.getage())

def setname(self, name):
    self.__name = name

def getname(self):
    return self.__name

#给类绑定方法
Temple.setname= setname
Temple.getname = getname

temp2 = Temple()
temp2.setname('name')
print(temp2.getname())
```

可以通过@property的方式，通过属性访问的方式就可以调用函数

``` python
class Definetion(object):
    def __init__(self, member):
        self.__member = member
    @property
    def member(self):
        print("call getter")
        return self.__member
    @member.setter
    def member(self, member):
        print("call setter")
        if not isinstance(member,int):
            raise TypeError("member must be int type")
        self.__member = member
    @member.deleter
    def member(self):
        print("call deleter")
        raise AttributeError("Cann't delete member")

definetioner = Definetion(3)
print(definetioner.member)
definetioner.member = 1024
print(definetioner.member)
```

将member分别实现为返回属性__member，设置__member，以及删除__member的函数。在每个member上添加对应格式的@property，@member.setter， @member.deleter。

通过属性访问的方式就可以调用对应的函数， definetioner.member返回__member值,  definetioner__member = 1024调用设置__member的函数。deleter definetioner.member调用的是删除函数。


## 实战：用pygame库做一个打飞机的小游戏

`pygame是python的一个做游戏的库`，安装方法自行百度。

实现子弹类

``` python
# 设置游戏屏幕大小
SCREEN_WIDTH = 480
SCREEN_HEIGHT = 800

# 子弹类
class Bullet(pygame.sprite.Sprite):
    def __init__(self, bullet_img, init_pos):
        pygame.sprite.Sprite.__init__(self)
        self.image = bullet_img
        self.rect = self.image.get_rect()
        self.rect.midbottom = init_pos
        self.speed = 10

    def move(self):
        self.rect.top -= self.speed
```
 

写玩家的飞机类

``` python
# 玩家飞机类
class Player(pygame.sprite.Sprite):
    def __init__(self, plane_img, player_rect, init_pos):
        pygame.sprite.Sprite.__init__(self)
        self.image = []                                 # 用来存储玩家飞机图片的列表
        for i in range(len(player_rect)):
            self.image.append(plane_img.subsurface(player_rect[i]).convert_alpha())
        self.rect = player_rect[0]                      # 初始化图片所在的矩形
        self.rect.topleft = init_pos                    # 初始化矩形的左上角坐标
        self.speed = 8                                  # 初始化玩家飞机速度，这里是一个确定的值
        self.bullets = pygame.sprite.Group()            # 玩家飞机所发射的子弹的集合
        self.img_index = 0                              # 玩家飞机图片索引
        self.is_hit = False                             # 玩家是否被击中
```
 

实现飞机类的几个功能函数

``` python
# 发射子弹
    def shoot(self, bullet_img):
        bullet = Bullet(bullet_img, self.rect.midtop)
        self.bullets.add(bullet)

    # 向上移动，需要判断边界
    def moveUp(self):
        if self.rect.top <= 0:
            self.rect.top = 0
        else:
            self.rect.top -= self.speed

    # 向下移动，需要判断边界
    def moveDown(self):
        if self.rect.top >= SCREEN_HEIGHT - self.rect.height:
            self.rect.top = SCREEN_HEIGHT - self.rect.height
        else:
            self.rect.top += self.speed

    # 向左移动，需要判断边界
    def moveLeft(self):
        if self.rect.left <= 0:
            self.rect.left = 0
        else:
            self.rect.left -= self.speed

    # 向右移动，需要判断边界        
    def moveRight(self):
        if self.rect.left >= SCREEN_WIDTH - self.rect.width:
            self.rect.left = SCREEN_WIDTH - self.rect.width
        else:
            self.rect.left += self.speed
```
实现敌方飞机类

``` python
# 敌机类
class Enemy(pygame.sprite.Sprite):
    def __init__(self, enemy_img, enemy_down_imgs, init_pos):
       pygame.sprite.Sprite.__init__(self)
       self.image = enemy_img
       self.rect = self.image.get_rect()
       self.rect.topleft = init_pos
       self.down_imgs = enemy_down_imgs
       self.speed = 2
       self.down_index = 0

    # 敌机移动，边界判断及删除在游戏主循环里处理
    def move(self):
        self.rect.top += self.speed
```

实现游戏主逻辑

``` python
# 初始化 pygame
pygame.init()

# 设置游戏界面大小、背景图片及标题
# 游戏界面像素大小
screen = pygame.display.set_mode((SCREEN_WIDTH, SCREEN_HEIGHT))

# 游戏界面标题
pygame.display.set_caption('飞机大战')

# 背景图
background = pygame.image.load('resources/image/background.png').convert()

# Game Over 的背景图
game_over = pygame.image.load('resources/image/gameover.png')

# 飞机及子弹图片集合
plane_img = pygame.image.load('resources/image/shoot.png')
```
由于资源采用大图的方式，敌机和飞机，子弹都绘制在一站图片上，需要裁剪，pygame提供裁剪函数

``` python
# 设置玩家飞机不同状态的图片列表，多张图片展示为动画效果
player_rect = []
player_rect.append(pygame.Rect(0, 99, 102, 126))        # 玩家飞机图片
player_rect.append(pygame.Rect(165, 360, 102, 126))
player_rect.append(pygame.Rect(165, 234, 102, 126))     # 玩家爆炸图片
player_rect.append(pygame.Rect(330, 624, 102, 126))
player_rect.append(pygame.Rect(330, 498, 102, 126))
player_rect.append(pygame.Rect(432, 624, 102, 126))
player_pos = [200, 600]
player = Player(plane_img, player_rect, player_pos)

# 子弹图片
bullet_rect = pygame.Rect(1004, 987, 9, 21)
bullet_img = plane_img.subsurface(bullet_rect)

# 敌机不同状态的图片列表，多张图片展示为动画效果
enemy1_rect = pygame.Rect(534, 612, 57, 43)
enemy1_img = plane_img.subsurface(enemy1_rect)
enemy1_down_imgs = []
enemy1_down_imgs.append(plane_img.subsurface(pygame.Rect(267, 347, 57, 43)))
enemy1_down_imgs.append(plane_img.subsurface(pygame.Rect(873, 697, 57, 43)))
enemy1_down_imgs.append(plane_img.subsurface(pygame.Rect(267, 296, 57, 43)))
enemy1_down_imgs.append(plane_img.subsurface(pygame.Rect(930, 697, 57, 43)))
```
管理敌机和敌机被击中的等对象

``` python
#存储敌机，管理多个对象
enemies1 = pygame.sprite.Group()

# 存储被击毁的飞机，用来渲染击毁动画
enemies_down = pygame.sprite.Group()

# 初始化射击及敌机移动频率
shoot_frequency = 0
enemy_frequency = 0

# 玩家飞机被击中后的效果处理
player_down_index = 16

# 初始化分数
score = 0

# 游戏循环帧率设置
clock = pygame.time.Clock()

# 判断游戏循环退出的参数
running = True
```

通过循环控制游戏逻辑，不断生成敌机和子弹，刷新场景等。

给大家个建议，也是忠告，pygame实现的游戏循环体中一定要捕捉事件消息，不然会因为死循环而一直卡顿，甚至崩溃。先实现循环体中事件捕捉

``` python
# 游戏主循环
while running:
    # 控制游戏最大帧率为 60
    clock.tick(60)

# 更新屏幕
    pygame.display.update()

    # 处理游戏退出
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            pygame.quit()
            exit()

    # 获取键盘事件（上下左右按键）
    key_pressed = pygame.key.get_pressed()

    # 处理键盘事件（移动飞机的位置）
    if key_pressed[K_w] or key_pressed[K_UP]:
        player.moveUp()
    if key_pressed[K_s] or key_pressed[K_DOWN]:
        player.moveDown()
    if key_pressed[K_a] or key_pressed[K_LEFT]:
        player.moveLeft()
    if key_pressed[K_d] or key_pressed[K_RIGHT]:
        player.moveRight()
```

 

在while running循环中添加子弹和敌机生成逻辑

``` python
# 生成子弹，需要控制发射频率
    # 首先判断玩家飞机没有被击中
    if not player.is_hit:
        if shoot_frequency % 15 == 0:
            player.shoot(bullet_img)
        shoot_frequency += 1
        if shoot_frequency >= 15:
            shoot_frequency = 0

    # 生成敌机，需要控制生成频率
    if enemy_frequency % 50 == 0:
        enemy1_pos = [random.randint(0, SCREEN_WIDTH - enemy1_rect.width), 0]
        enemy1 = Enemy(enemy1_img, enemy1_down_imgs, enemy1_pos)
        enemies1.add(enemy1)
    enemy_frequency += 1
    if enemy_frequency >= 100:
        enemy_frequency = 0
```

 

在while running循环中子弹和敌机移动逻辑

``` python
for bullet in player.bullets:
        # 以固定速度移动子弹
        bullet.move()
        # 移动出屏幕后删除子弹
        if bullet.rect.bottom < 0:
            player.bullets.remove(bullet)   

    for enemy in enemies1:
        #2. 移动敌机
        enemy.move()
        #3. 敌机与玩家飞机碰撞效果处理
        if pygame.sprite.collide_circle(enemy, player):
            enemies_down.add(enemy)
            enemies1.remove(enemy)
            player.is_hit = True
            break
        #4. 移动出屏幕后删除飞机    
        if enemy.rect.top < 0:
            enemies1.remove(enemy)

    #敌机被子弹击中效果处理
    # 将被击中的敌机对象添加到击毁敌机 Group 中，用来渲染击毁动画
    enemies1_down = pygame.sprite.groupcollide(enemies1, player.bullets, 1, 1)
    for enemy_down in enemies1_down:
        enemies_down.add(enemy_down)
```

 

在while running循环中添加自己飞机动态逻辑

``` python
# 绘制玩家飞机
    if not player.is_hit:
        screen.blit(player.image[player.img_index], player.rect)
        # 更换图片索引使飞机有动画效果
        player.img_index = shoot_frequency // 8
    else:
        # 玩家飞机被击中后的效果处理
        player.img_index = player_down_index // 8
        screen.blit(player.image[player.img_index], player.rect)
        player_down_index += 1
        if player_down_index > 47:
            # 击中效果处理完成后游戏结束
            running = False

    # 敌机被子弹击中效果显示
    for enemy_down in enemies_down:
        if enemy_down.down_index == 0:
            pass
        if enemy_down.down_index > 7:
            enemies_down.remove(enemy_down)
            score += 1000
            continue
        screen.blit(enemy_down.down_imgs[enemy_down.down_index // 2], enemy_down.rect)
        enemy_down.down_index += 1
```

效果显示：
![1.png](1.png)
源码下载地址：[打飞机小游戏python](https://github.com/secondtonone1/python-/tree/master/plane)

谢谢关注我的公众号：
![1.jpg](1.jpg)