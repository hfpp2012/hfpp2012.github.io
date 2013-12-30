---
layout: post
title: "ubuntu下sublime text 3输入中文"
date: 2013-12-30 21:11
comments: true
categories: linux editor subl
---

+ [sublime text 3](http://www.sublimetext.com/3)

从官网下载的sublime text 3在linux下是无法输入中文的,但是有一个解决方法,它是通过编译一个c文件来实现的,然后加载它,最后sublime text 3就可以输入中文了

我的环境

+ ubuntu 13.04
+ fcitx 4.2.7
+ sublime text 3(stable build 3059)

<!-- more -->

### 安装必要的软件环境

``` bash
sudo apt-get install build-essential
sudo apt-get install libgtk2.0-dev
```

### 复制该文件存在家目录下~/sublime-imfix.c

``` c sublime-imfix.c
/*
sublime-imfix.c
Use LD_PRELOAD to interpose some function to fix sublime input method support for linux.
By Cjacker Huang
 
gcc -shared -o libsublime-imfix.so sublime-imfix.c `pkg-config --libs --cflags gtk+-2.0` -fPIC
LD_PRELOAD=./libsublime-imfix.so subl
*/
#include <gtk/gtk.h>
#include <gdk/gdkx.h>
typedef GdkSegment GdkRegionBox;
 
struct _GdkRegion
{
  long size;
  long numRects;
  GdkRegionBox *rects;
  GdkRegionBox extents;
};
 
GtkIMContext *local_context;
 
void
gdk_region_get_clipbox (const GdkRegion *region,
            GdkRectangle    *rectangle)
{
  g_return_if_fail (region != NULL);
  g_return_if_fail (rectangle != NULL);
 
  rectangle->x = region->extents.x1;
  rectangle->y = region->extents.y1;
  rectangle->width = region->extents.x2 - region->extents.x1;
  rectangle->height = region->extents.y2 - region->extents.y1;
  GdkRectangle rect;
  rect.x = rectangle->x;
  rect.y = rectangle->y;
  rect.width = 0;
  rect.height = rectangle->height;
  //The caret width is 2;
  //Maybe sometimes we will make a mistake, but for most of the time, it should be the caret.
  if(rectangle->width == 2 && GTK_IS_IM_CONTEXT(local_context)) {
        gtk_im_context_set_cursor_location(local_context, rectangle);
  }
}
 
//this is needed, for example, if you input something in file dialog and return back the edit area
//context will lost, so here we set it again.
 
static GdkFilterReturn event_filter (GdkXEvent *xevent, GdkEvent *event, gpointer im_context)
{
    XEvent *xev = (XEvent *)xevent;
    if(xev->type == KeyRelease && GTK_IS_IM_CONTEXT(im_context)) {
       GdkWindow * win = g_object_get_data(G_OBJECT(im_context),"window");
       if(GDK_IS_WINDOW(win))
         gtk_im_context_set_client_window(im_context, win);
    }
    return GDK_FILTER_CONTINUE;
}
 
void gtk_im_context_set_client_window (GtkIMContext *context,
          GdkWindow    *window)
{
  GtkIMContextClass *klass;
  g_return_if_fail (GTK_IS_IM_CONTEXT (context));
  klass = GTK_IM_CONTEXT_GET_CLASS (context);
  if (klass->set_client_window)
    klass->set_client_window (context, window);
 
  if(!GDK_IS_WINDOW (window))
    return;
  g_object_set_data(G_OBJECT(context),"window",window);
  int width = gdk_window_get_width(window);
  int height = gdk_window_get_height(window);
  if(width != 0 && height !=0) {
    gtk_im_context_focus_in(context);
    local_context = context;
  }
  gdk_window_add_filter (window, event_filter, context);
}

```

### 将sublime-imfix.c编译成共享内库

``` bash
gcc -shared -o libsublime-imfix.so sublime-imfix.c `pkg-config --libs --cflags gtk+-2.0` -fPIC
```

它会生成libsublime-imfix.so的库文件

### 修改.bashrc加载该内库,这样命令subl开的sublime text输入中文就没问题了

``` bash ~/.bashrc
export LD_PRELOAD=/home/yinsigan/libsublime-imfix.so subl
```

``` bash
source ~/.bashrc
```

{% img /images/sublimelinux/c.gif %}

/home/yinsigan要替换成你自己的家目录,这里使用绝对路径

### 修改 /usr/share/applications/sublime_text.desktop 为

```
[Desktop Entry]
[...]
Exec=env LD_PRELOAD=/home/yinsigan/libsublime-imfix.so /opt/sublime_text/sublime_text %F
[...]
 
[Desktop Action Window]
[...]
Exec=env LD_PRELOAD=/home/yinsigan/libsublime-imfix.so /opt/sublime_text/sublime_text -n
[...]
 
[Desktop Action Document]
[...]
Exec=env LD_PRELOAD=/home/yinsigan/libsublime-imfix.so /opt/sublime_text/sublime_text --command new_file
[...]
```

这样在桌面上点击sublime text的图标开的sublime text也可以输入中文的

### 最后是效果图

{% img /images/sublimelinux/sublime.png %}

