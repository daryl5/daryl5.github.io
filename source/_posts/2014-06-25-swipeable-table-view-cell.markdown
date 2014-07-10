---
layout: post
title: "Make A Swipeable TableviewCell"
date: 2014-06-25 19:59:06 +0800
comments: true
categories: iOS
---
来自RW的博文[How To Make A Swipeable Table View Cell With Actions – Without Going Nuts With Scroll Views](http://www.raywenderlich.com/62435/make-swipeable-table-view-cell-actions-without-going-nuts-scroll-views)。  
<!--more-->
###1.UITableViewCell结构解析
`给view添加背景色`和`recursiveDescription`都可以用来解析控件结构。
```objective-c
#ifdef DEBUG
NSLog(@"Cell recursive description:\n\n%@\n\n", [cell performSelector:@selector(recursiveDescription)]);
#endif
```
iOS7上tableViewCell的结构如下：
```objective-c
TableViewCell
    CellScrollView
        //SeparatorView
        Button                 disclosure button
            ImageView          button's image
        ContentView
            Label              cell text label
        //SeparatorView
```
删除时候结构如下：
```objective-c
TableViewCell
    CellScrollView
        DeleteConfirmationView
            Confirmation Button
                Label          "Delete"
        //SeparatorView
        ContentView
            Label
        //Separator View
        Button
            ImageView
```
iOS8上如下，对比iOS7主要是少了`UITableViewCellScrollView`，这是个UIScrollView的实例：
```objective-c
TableViewCell
    ContentView
        Label                  cell text label
    //SeparatorView
    Button
        ImageView
```
根据官方文档，自定义cell只能在ContentView上添加subview。我们要做的，就是参照苹果实现滑动展示删除按钮的思路，在ContentView上另外实现一套，然后把原本的删除禁掉。
###2.一个Swipeable TableViewCell的必要成分
由上一部分最后一段可以知道，我们需要一些`需要展示的UIButton`，然后一个位于这些按钮之上的`Container view`来显示所有内容，然后需要一个UIScrollView或者`UIPanGestureRecognizer`来显示或者隐藏按钮，这里我们用后者。最后，`需要展示cell内容的views`。然后把这些内容都贴到cell的ContentView上。这里有点拗口，对应上面iOS7的cell结构。  
需要禁掉系统cell的滑动删除：
```objective-c
- (BOOL)tableView:(UITableView *)tableView canEditRowAtIndexPath:(NSIndexPath *)indexPath {
    return NO;
}
```
###3.建立自定义cell
直接在storyboard中编辑prototypecell，然后绑定cell所属类，并在tableview代理中返回相应的类型。这里非常坑，半天显示不出来两个按钮，基础太差代码写的太少了，哎。原因见[Xcode 5.1 UITableView in UIViewController - Custom TableViewCell Outlets are nil](http://stackoverflow.com/questions/22352587/xcode-5-1-uitableview-in-uiviewcontroller-custom-tableviewcell-outlets-are-nil)。具体原因是：  

1. 如果在xib或者storyboard中建立cell了，那么直接deque就能获得cell；
2. 如果要用自定义的cell，那么就要先registerclass，然后重用的时候deque；这时候虽然没有xib，但是运行时也知道如何去建立cell，就用刚注册过的类建立。

未完待续...




















