# 红黑树

​		**红黑树**（英语：Red–black tree）是一种自平衡二叉查找树,红黑树的结构复杂，但它的操作有着良好的最坏情况运行时间，查找、插入和删除时间复杂度为O(logn)。

​		其实红黑树也是二叉查找树的一种，二叉查找树在最坏的情况下可能会变成一个链表（当所有 节点按从小到大的顺序依次插入后）；这样树的平衡性被破坏，要提高查询效率需要维持这种平衡和降低树的高度，而其中可性的做法就是用策略在每次修改树的内容之后进行结构调整，使其保持一定的平衡条件；而红黑树就是其中一员。

## 优点

​	红黑树在插入时间、删除时间和查找时间提供最好可能的最坏情况担保，多数用时间敏感或者实时性强的应用中。

​	红黑树相对于AVL树来说，牺牲了部分平衡性以换取插入和删除操作时少量的旋转操作，整体来说性能要优于AVL树

## 性质

​	红黑树是每个节点都带有颜色属性的二叉查找树，颜色为红色或黑色。还需遵循以下规则：

1. 节点分为红色或者黑色
2. 根节点比必须是黑色
3. 所有叶子都是黑色（叶子是NIL节点，或者称为空（NULL）叶子）
4. 每个红色节点必须有两个黑色的子节点。（从每个叶子到根的所有路径上不能有两个连续的红色节点。）
5. 从任一节点到其每个叶子的所有简单路径都包含相同数目的黑色节点。

**红黑树的图例：**

![Red-black_tree_example.svg](./img/Red-black_tree_example.svg.png )





## 操作

​			在红黑树上进行插入操作和删除操作会导致不再符合红黑树的性质，因此需对红黑树进行性质的恢复，但必须做到少量![o(logn)](/Users/wangxueke/learn/data-structure_learn/img/o(logn).svg)的颜色变更,而且不超过三次树旋转（对于插入操作是两次）；由此可以看出红黑树的插入和删除复杂，但是操作时间仍然可以保持O(logn)

### 树旋转

​		**树旋转**是不影响元素顺序，但会改变树的结构，将一个节点上移、一个节点下移；

​		左孩子旋转到父节点的位置为**右旋**；

​		右孩子旋转到父节点的位置为**左旋**；				

​		![Tree_rotation](./img/Tree_rotation.svg)

红黑树的左旋代码如下：

```c
static void __rb_rotate_left(struct rb_node *node, struct rb_root *root)
{
	struct rb_node *right = node->rb_right;//取出node的右节点
	struct rb_node *parent = rb_parent(node);//取出node的父亲节点

	if ((node->rb_right = right->rb_left))//node的右子节点指向right的左子节点时
		rb_set_parent(right->rb_left, node);//并将right的左子节点与right脱离，right的左子节点的父亲节点指向node
	right->rb_left = node;//将right的左子节点重新指向node
	/****  到现在就完成了，node右侧树与node脱离，接下来就是将脱离的右侧树重新连接parent  *****/
	rb_set_parent(right, parent);//将right的父亲节点指向node父亲节点
  
	if (parent)//node非根节点
	{
		if (node == parent->rb_left)//判断node所在parent的方向并将parent对应的节点重新指向right
			parent->rb_left = right;
		else
			parent->rb_right = right;
	}
	else //node是根节点，那将right设置为根节点
		root->rb_node = right;
	rb_set_parent(node, right);//最后将node的父亲节点设置为right
}
```

红黑树的右旋代码如下：

```c
static void __rb_rotate_right(struct rb_node *node, struct rb_root *root)
{
	struct rb_node *left = node->rb_left;//取出node的左节点
	struct rb_node *parent = rb_parent(node);//取出node的父亲节点

	if ((node->rb_left = left->rb_right))//将node的左子节点指向left的右子节点
		rb_set_parent(left->rb_right, node);//将left的右子节点的父亲节点指向node
	left->rb_right = node;//将left的右子节点重新指向node
	/****  到现在就完成了，node左侧树与node脱离，接下来就是将脱离的左侧树重新连接parent  *****/
	rb_set_parent(left, parent);//将left的父亲节点指向node父亲节点

	if (parent)//node非根节点
	{
		if (node == parent->rb_right)//判断node所在parent的方向并将parent对应的节点重新指向left
			parent->rb_right = left;
		else
			parent->rb_left = left;
	}
	else//node是根节点，那将left设置为根节点
		root->rb_node = left;
	rb_set_parent(node, left);//最后将node的父亲节点设置为left
}
```

### 颜色调换

​		保持红黑树的性质，结点不能乱挪，还得靠变色了；具体怎么样来调整颜色还要看插入和删除的情况来定，在说明插入和删除操作的时候来详细说明，所以现在只需记住**红黑树总是通过旋转和变色达到自平衡**。

### 插入

​		首先在增加节点时，将其标记成红色，如果设置为黑色就会导致根到叶子的路径上有一条路上，多一个额外的黑节点，这个是很难调整的。但是设为红色节点后，可能会导致出现两个连续红色节点的冲突，那么可以通过颜色调换（color flips）和树旋转来调整。

流程图如下：

![rb_tree_install](./img/rb_tree_install.png)

实现代码

```c
void rb_insert_color(struct rb_node *node, struct rb_root *root)//已经查找到相应的位置，进行重新调整平衡
{
	struct rb_node *parent, *gparent;

	while ((parent = rb_parent(node)) && rb_is_red(parent))//取出父亲节点，并父亲节点是红色的时候
	{
		gparent = rb_parent(parent);//获取祖父节点

		if (parent == gparent->rb_left)//如果父亲节点是在祖父节点左侧
		{
			{
				register struct rb_node *uncle = gparent->rb_right;
				if (uncle && rb_is_red(uncle))//存在在叔叔节点，父亲节点和叔叔节点都为红色，当父亲节点在祖父节点左侧
				{
					rb_set_black(uncle);//将父亲节点和叔叔节点都置为黑色
					rb_set_black(parent);
					rb_set_red(gparent);//再将祖父节点置为红色 （保持性质5）
					node = gparent;//将祖父节点当成是新加入的节点进行各种情形的檢查
					continue;
				}
			}

      //情形4、当父亲节点为红色，叔叔节点缺失或为黑色，新节点为其父亲节点的右侧时，其父亲节点在祖父节点左侧
			if (parent->rb_right == node)
			{
				register struct rb_node *tmp;
				__rb_rotate_left(parent, root);//针对父亲节点进行一次左旋转，并将新节点位置和其父亲节点位置进行交换
				tmp = parent;
				parent = node;
				node = tmp;
			}
      //完成情况4，还需解决仍然失效的性质4；
      //情形5、将父亲节点置为黑色，在将祖父节点置为红色，并针对祖父节点进行一次右旋转，使其结果满足性质4
			rb_set_black(parent);
			rb_set_red(gparent);
			__rb_rotate_right(gparent, root);
		} else {//如果父亲节点是在祖父节点右侧，
     
			{
				register struct rb_node *uncle = gparent->rb_left;
				if (uncle && rb_is_red(uncle))//存在在叔叔节点，父亲节点和叔叔节点都为红色，当父亲节点在祖父节点右侧
				{
					rb_set_black(uncle);//将父亲节点和叔叔节点都置为黑色
					rb_set_black(parent);
					rb_set_red(gparent);//再将祖父节点置为红色 （保持性质5）
					node = gparent;//将祖父节点当成是新加入的节点进行各种情形的檢查
					continue;
				}
			}
 			//因为父亲节点是在祖父节点右侧，情形4和情形5中的左和右应当对调。
			if (parent->rb_left == node)
			{
				register struct rb_node *tmp;
				__rb_rotate_right(parent, root);
				tmp = parent;
				parent = node;
				node = tmp;
			}

			rb_set_black(parent);
			rb_set_red(gparent);
			__rb_rotate_left(gparent, root);
		}
	}

	rb_set_black(root->rb_node);//将根节点设置为黑色（保持性质2）
}
```

#### 情况一

​		新节点在树的根节点上，只需将根节点重置为黑色，保持性质2。

#### 情况二

​		新节点的父亲节点为黑色，所以性质4没有失效，由于每次新增加的节点都为红色，尽管新节点存在两个黑色叶子节点，但是通過它的每个子节点的路径和它所取代的黑色的叶子的路径具有相同数目的黑色节点，所以满足性质5

#### 情况三

​		存在在叔叔节点，父亲节点和叔叔节点都为红色，将父亲节点和叔叔节点都置为黑色，再将祖父节点置为红色 ，这样就保持了性质5；紅色的祖父节点可能是根节点，这就违反了性质2，也有可能祖父节点的父节点是紅色的，这就违反了性质4；这种情况下需将祖父节点当成是新加入的节点进行各种情形的檢查。

![](./img/Red-black_tree_insert_case_3.png)

**注意：***<u>在余下的情形下，假设父亲节点是在祖父节点左侧，如果父亲节点是在祖父节点右侧，情形4和情形5中的左和右应当对调。</u>*

#### 情况四

​		当父亲节点为红色，叔叔节点缺失或为黑色，新节点为其父亲节点的右侧时，其父亲节点在祖父节点左侧；针对父亲节点进行一次左旋转，并将新节点位置和其父亲节点位置进行交换；完成这种情况后，还需解决仍然失效的性质4，那就进行情况5处理；

![](./img/Red-black_tree_insert_case_4.png)

#### 情况5

​		当父亲节点为红色，叔叔节点缺失或为黑色，新节点为其父亲节点的左侧时，其父亲节点在祖父节点左侧；将父亲节点置为黑色，在将祖父节点置为红色，并针对祖父节点进行一次右旋转，使其结果满足性质4；性质5也仍然保持滿足，因为通过这三個节点中任何一個的所有路径以前都通过祖父节点，現在它們都通过以前的父节点。在各自的情形下，这都是三个节点中唯一的黑色节点。

![](/Users/wangxueke/learn/data-structure_learn/img/Red-black_tree_insert_case_5.png)

