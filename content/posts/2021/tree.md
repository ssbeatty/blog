---
title: 邓书树结构
date: 2021-01-26T09:43:05+08:00
lastmod: 2021-01-26T10:43:05+08:00
author: sasaba
cover: /img/邓书树结构.jpg
images:
  - /img/邓书树结构.jpg
categories:
  - 数据结构
tags:
  - 算法
  - 数据结构
draft: true
---

邓书树结构C++实现，可以参考遍历树结构的方法。

<!--more-->

## 二叉树

> 树是一种连通无环图。
>
> 树的高度是指最深的叶子的高度。
>
> n个顶点的树有n-1个边。

**树节点**

```c++
#define BinNodePosi(T) BinNode<T>* //节点位置
#define stature(p) ((p) ? (p)->height : -1) //节点高度（与“空树高度为-1”的约定相统一）
typedef enum { RB_RED, RB_BLACK} RBColor; //节点颜色

template <typename T> struct BinNode { //二叉树节点模板类
// 成员（为简化描述起见统一开放，读者可根据需要进一步封装）
   T data; //数值
   BinNodePosi(T) parent; BinNodePosi(T) lc; BinNodePosi(T) rc; //父节点及左、右孩子
   int height; //高度（通用）
   int npl; //Null Path Length（左式堆，也可直接用height代替）
   RBColor color; //颜色（红黑树）
// 构造函数
   BinNode() :
      parent ( NULL ), lc ( NULL ), rc ( NULL ), height ( 0 ), npl ( 1 ), color ( RB_RED ) { }
   BinNode ( T e, BinNodePosi(T) p = NULL, BinNodePosi(T) lc = NULL, BinNodePosi(T) rc = NULL,
             int h = 0, int l = 1, RBColor c = RB_RED ) :
      data ( e ), parent ( p ), lc ( lc ), rc ( rc ), height ( h ), npl ( l ), color ( c ) { }
// 操作接口
   int size(); //统计当前节点后代总数，亦即以其为根的子树的规模
   BinNodePosi(T) insertAsLC ( T const& ); //作为当前节点的左孩子插入新节点
   BinNodePosi(T) insertAsRC ( T const& ); //作为当前节点的右孩子插入新节点
   BinNodePosi(T) succ(); //取当前节点的直接后继
   template <typename VST> void travLevel ( VST& ); //子树层次遍历
   template <typename VST> void travPre ( VST& ); //子树先序遍历
   template <typename VST> void travIn ( VST& ); //子树中序遍历
   template <typename VST> void travPost ( VST& ); //子树后序遍历
// 比较器、判等器（各列其一，其余自行补充）
   bool operator< ( BinNode const& bn ) { return data < bn.data; } //小于
   bool operator== ( BinNode const& bn ) { return data == bn.data; } //等于
   BinNodePosi(T) zig(); //顺时针旋转
   BinNodePosi(T) zag(); //逆时针旋转
   BinNodePosi(T) balance(); //完全平衡化
   BinNodePosi(T) imitate( const BinNodePosi(T) );
};

```

**接口**

```c++
// 插入
template <typename T> BinNodePosi(T) BinNode<T>::insertAsLC ( T const& e )
{ return lc = new BinNode ( e, this ); } //将e作为当前节点的左孩子插入二叉树

template <typename T> BinNodePosi(T) BinNode<T>::insertAsRC ( T const& e )
{ return rc = new BinNode ( e, this ); } //将e作为当前节点的右孩子插入二叉树

// size
template <typename T> int BinNode<T>::size() { //统计当前节点后代总数，即以其为根的子树规模
   int s = 1; //计入本身
   if ( lc ) s += lc->size(); //递归计入左子树规模
   if ( rc ) s += rc->size(); //递归计入右子树规模
   return s;
}
```



**二叉树**

```c++
#include "BinNode.h" //引入二叉树节点类
template <typename T> class BinTree { //二叉树模板类
protected:
   int _size; BinNodePosi(T) _root; //规模、根节点
   virtual int updateHeight ( BinNodePosi(T) x ); //更新节点x的高度
   void updateHeightAbove ( BinNodePosi(T) x ); //更新节点x及其祖先的高度
public:
   BinTree() : _size ( 0 ), _root ( NULL ) { } //构造函数
   ~BinTree() { if ( 0 < _size ) remove ( _root ); } //析构函数
   int size() const { return _size; } //规模
   bool empty() const { return !_root; } //判空
   BinNodePosi(T) root() const { return _root; } //树根
   BinNodePosi(T) insertAsRoot ( T const& e ); //插入根节点
   BinNodePosi(T) insertAsLC ( BinNodePosi(T) x, T const& e ); //e作为x的左孩子（原无）插入
   BinNodePosi(T) insertAsRC ( BinNodePosi(T) x, T const& e ); //e作为x的右孩子（原无）插入
   BinNodePosi(T) attachAsLC ( BinNodePosi(T) x, BinTree<T>* &T ); //T作为x左子树接入
   BinNodePosi(T) attachAsRC ( BinNodePosi(T) x, BinTree<T>* &T ); //T作为x右子树接入
   int remove ( BinNodePosi(T) x ); //删除以位置x处节点为根的子树，返回该子树原先的规模
   BinTree<T>* secede ( BinNodePosi(T) x ); //将子树x从当前树中摘除，并将其转换为一棵独立子树
   template <typename VST> //操作器
   void travLevel ( VST& visit ) { if ( _root ) _root->travLevel ( visit ); } //层次遍历
   template <typename VST> //操作器
   void travPre ( VST& visit ) { if ( _root ) _root->travPre ( visit ); } //先序遍历
   template <typename VST> //操作器
   void travIn ( VST& visit ) { if ( _root ) _root->travIn ( visit ); } //中序遍历
   template <typename VST> //操作器
   void travPost ( VST& visit ) { if ( _root ) _root->travPost ( visit ); } //后序遍历
   bool operator< ( BinTree<T> const& t ) //比较器（其余自行补充）
   { return _root && t._root && lt ( _root, t._root ); }
   bool operator== ( BinTree<T> const& t ) //判等器
   { return _root && t._root && ( _root == t._root ); }
}; //BinTree
```

**接口**

```c++
// stature(p) ((p) ? (p)->height : -1)
// 更新高度
template <typename T> int BinTree<T>::updateHeight ( BinNodePosi(T) x ) //更新节点x高度
{ return x->height = 1 + __max ( stature ( x->lc ), stature ( x->rc ) ); } //具体规则，因树而异

template <typename T> void BinTree<T>::updateHeightAbove ( BinNodePosi(T) x ) //更新高度
{ while ( x ) { updateHeight ( x ); x = x->parent; } } //从x出发，覆盖历代祖先。可优化

// insert
template <typename T> BinNodePosi(T) BinTree<T>::insertAsRoot ( T const& e )
{ _size = 1; return _root = new BinNode<T> ( e ); } //将e当作根节点插入空的二叉树

template <typename T> BinNodePosi(T) BinTree<T>::insertAsLC ( BinNodePosi(T) x, T const& e )
{ _size++; x->insertAsLC ( e ); updateHeightAbove ( x ); return x->lc; } //e插入为x的左孩子

template <typename T> BinNodePosi(T) BinTree<T>::insertAsRC ( BinNodePosi(T) x, T const& e )
{ _size++; x->insertAsRC ( e ); updateHeightAbove ( x ); return x->rc; } //e插入为x的右孩子
//insertAsLC()完全对称，在此省略

```

> 先序 中序 后序事实上是指根节点在顺序的位置，如果先遍历根节点则先序，左-->根-->右 则中序。

**先序遍历**

```c++
//visit 是函数的指针用来处理节点
template <typename T, typename VST> //元素类型、操作器
void travPre_R ( BinNodePosi(T) x, VST& visit ) { //二叉树先序遍历算法（递归版）
   if ( !x ) return;
   visit ( x->data );
   travPre_R ( x->lc, visit );
   travPre_R ( x->rc, visit );
}

//上面的改为迭代版本 栈实现
template <typename T, typename VST> //元素类型、操作器
void travPre_I1 ( BinNodePosi(T) x, VST& visit ) { //二叉树先序遍历算法（迭代版#1）
   Stack<BinNodePosi(T)> S; //辅助栈
   if ( x ) S.push ( x ); //根节点入栈
   while ( !S.empty() ) { //在栈变空之前反复循环
      x = S.pop(); visit ( x->data ); //弹出并访问当前节点，其非空孩子的入栈次序为先右后左
      if ( HasRChild ( *x ) ) S.push ( x->rc ); if ( HasLChild ( *x ) ) S.push ( x->lc );
   }
}

//实现版本2
//从当前节点出发，沿左分支不断深入，直至没有左分支的节点；沿途节点遇到后立即访问
template <typename T, typename VST> //元素类型、操作器
static void visitAlongVine ( BinNodePosi(T) x, VST& visit, Stack<BinNodePosi(T)>& S ) {
   while ( x ) {
      visit ( x->data ); //访问当前节点
      S.push ( x->rc ); //右孩子入栈暂存（可优化：通过判断，避免空的右孩子入栈）
      x = x->lc;  //沿左分支深入一层
   }
}
template <typename T, typename VST> //元素类型、操作器
void travPre_I2 ( BinNodePosi(T) x, VST& visit ) { //二叉树先序遍历算法（迭代版#2）
   Stack<BinNodePosi(T)> S; //辅助栈
   while ( true ) {
      visitAlongVine ( x, visit, S ); //从当前节点出发，逐批访问
      if ( S.empty() ) break; //直到栈空
      x = S.pop(); //弹出下一批的起点
   }
}
```

**中序遍历**

```c++
//递归
template <typename T, typename VST> //元素类型、操作器
void travIn_R ( BinNodePosi(T) x, VST& visit ) { //二叉树中序遍历算法（递归版）
   if ( !x ) return;
   travIn_R ( x->lc, visit );
   visit ( x->data );
   travIn_R ( x->rc, visit );
}

//实现1
template <typename T> //从当前节点出发，沿左分支不断深入，直至没有左分支的节点
static void goAlongVine ( BinNodePosi(T) x, Stack<BinNodePosi(T)>& S ) {
   while ( x ) { S.push ( x ); x = x->lc; } //当前节点入栈后随即向左侧分支深入，迭代直到无左孩子
}
template <typename T, typename VST> //元素类型、操作器
void travIn_I1 ( BinNodePosi(T) x, VST& visit ) { //二叉树中序遍历算法（迭代版#1）
   Stack<BinNodePosi(T)> S; //辅助栈
   while ( true ) {
      goAlongVine ( x, S ); //从当前节点出发，逐批入栈
      if ( S.empty() ) break; //直至所有节点处理完毕
      x = S.pop(); visit ( x->data ); //弹出栈顶节点并访问之
      x = x->rc; //转向右子树
   }
}

//实现2
template <typename T, typename VST> //元素类型、操作器
void travIn_I2 ( BinNodePosi(T) x, VST& visit ) { //二叉树中序遍历算法（迭代版#2）
   Stack<BinNodePosi(T)> S; //辅助栈
   while ( true )
      if ( x ) {
         S.push ( x ); //根节点进栈
         x = x->lc; //深入遍历左子树
      } else if ( !S.empty() ) {
         x = S.pop(); //尚未访问的最低祖先节点退栈
         visit ( x->data ); //访问该祖先节点
         x = x->rc; //遍历祖先的右子树
      } else
         break; //遍历完成
}

//实现3
template <typename T, typename VST> //元素类型、操作器
void travIn_I3 ( BinNodePosi(T) x, VST& visit ) { //二叉树中序遍历算法（迭代版#3，无需辅助栈）
   bool backtrack = false; //前一步是否刚从左子树回溯——省去栈，仅O(1)辅助空间
   while ( true )
      if ( !backtrack && HasLChild ( *x ) ) //若有左子树且不是刚刚回溯，则
         x = x->lc; //深入遍历左子树
      else { //否则——无左子树或刚刚回溯（相当于无左子树）
         visit ( x->data ); //访问该节点
         if ( HasRChild ( *x ) ) { //若其右子树非空，则
            x = x->rc; //深入右子树继续遍历
            backtrack = false; //并关闭回溯标志
         } else { //若右子树空，则
            if ( ! ( x = x->succ() ) ) break; //回溯（含抵达末节点时的退出返回）
            backtrack = true; //并设置回溯标志
         }
      }
}

//实现4
template <typename T, typename VST> //元素类型、操作器
void travIn_I4 ( BinNodePosi(T) x, VST& visit ) { //二叉树中序遍历（迭代版#4，无需栈或标志位）
   while ( true )
      if ( HasLChild ( *x ) ) //若有左子树，则
         x = x->lc; //深入遍历左子树
      else { //否则
         visit ( x->data ); //访问当前节点，并
         while ( !HasRChild ( *x ) ) //不断地在无右分支处
            if ( ! ( x = x->succ() ) ) return; //回溯至直接后继（在没有后继的末节点处，直接退出）
            else visit ( x->data ); //访问新的当前节点
         x = x->rc; //（直至有右分支处）转向非空的右子树
      }
}
```

**后序遍历**

```c++
//递归
template <typename T, typename VST> //元素类型、操作器
void travPost_R ( BinNodePosi(T) x, VST& visit ) { //二叉树后序遍历算法（递归版）
   if ( !x ) return;
   travPost_R ( x->lc, visit );
   travPost_R ( x->rc, visit );
   visit ( x->data );
}

// 迭代版
template <typename T> //在以S栈顶节点为根的子树中，找到最高左侧可见叶节点
static void gotoLeftmostLeaf ( Stack<BinNodePosi(T)>& S ) { //沿途所遇节点依次入栈
   while ( BinNodePosi(T) x = S.top() ) //自顶而下，反复检查当前节点（即栈顶）
      if ( HasLChild ( *x ) ) { //尽可能向左
         if ( HasRChild ( *x ) ) S.push ( x->rc ); //若有右孩子，优先入栈
         S.push ( x->lc ); //然后才转至左孩子
      } else //实不得已
         S.push ( x->rc ); //才向右
   S.pop(); //返回之前，弹出栈顶的空节点
}

template <typename T, typename VST>
void travPost_I ( BinNodePosi(T) x, VST& visit ) { //二叉树的后序遍历（迭代版）
   Stack<BinNodePosi(T)> S; //辅助栈
   if ( x ) S.push ( x ); //根节点入栈
   while ( !S.empty() ) { //x始终为当前节点
      if ( S.top() != x->parent ) ////若栈顶非x之父（而为右兄）
         gotoLeftmostLeaf ( S ); //则在其右兄子树中找到HLVFL（相当于递归深入）
      x = S.pop(); visit ( x->data ); //弹出栈顶（即前一节点之后继），并访问之
   }
}
```



**层次遍历**

```c++
/*DSA*/#include "queue/queue.h" //引入队列
template <typename T> template <typename VST> //元素类型、操作器
void BinNode<T>::travLevel ( VST& visit ) { //二叉树层次遍历算法
   Queue<BinNodePosi(T)> Q; //辅助队列
   Q.enqueue ( this ); //根节点入队
   while ( !Q.empty() ) { //在队列再次变空之前，反复迭代
      BinNodePosi(T) x = Q.dequeue(); visit ( x->data ); //取出队首节点并访问之
      if ( HasLChild ( *x ) ) Q.enqueue ( x->lc ); //左孩子入队
      if ( HasRChild ( *x ) ) Q.enqueue ( x->rc ); //右孩子入队
   }
}
```





















