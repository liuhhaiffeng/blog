---
title: 'traversing tree in preorder inorder postorder and level order'
comments: true
date: 2017-03-25 21:32:06
tags: [编程, c]
categories: 技术分享
---

> Tree as a very common and useful data structure is worth extensive study. Years after I took the Data-structure lesson when I was in school, for the last few days, I regrab the knowledge of how to implement and tranversing a binary tree in servral ways. Besides of the old ways such as preorder/indorder/postorder to traversing a tree, I learnt new knowledge of ways to traversing a tree: level order traversing. Further more, I knew how to print the graph of the tree using graphviz.

<center>![image_1bc2d7db4nop98t1ukvunk61p9.png-29.5kB][1] </center>
<center>Fig.1 a binary tree</center>

The graph above is drawn by `graphviz` using the output data by our traverse program list below. You can easily figure out the several ways of how to traverse such a binary tree:

```
root first:	5	3	2	1	4	6	7	8	
root secnd:	1	2	3	4	5	6	7	8	
root last:	1	2	4	3	8	7	6	5	
level out:	5	3	6	2	4	7	1	8	
```

As you might have seen, the root-second  or the inorder traverse way of such a binary search tree is ordered. In the following section, I will try to (1) constructing a binary tree using basic C structs and pointers. (2) implement different ways of how to traversing a tree.

Constructing a binary tree
---------

In many books, the `Tree` and  `Node` structure often share the same defination. However, it is much more clear if you store them seperatly in two different struct. Therefore, you can store additional info in `Tree` structure inaddtion to `Node` structure. Currently, the `Tree` structure only has a member `struct _node *` which point to the root of the tree. So, I decided to use the following basic data-structure as the fondation brick of my implementation.
```c
typedef struct _node{
    int value;
    struct _node  * left;
    struct _node  * right;
} Node;

typedef struct _tree{
    struct _node  *root;
} Tree;

Tree * new_tree()
{
    Tree *tree;

    tree = (Tree *)malloc(sizeof(*tree));
    tree->root = NULL;

    return tree;
}
```
Function `Tree * new_tree()` return an empty tree whose root point to `NULL`.

Inserting in to a binary tree
---------
Here we'll insert values in order so we have a binary search tree. In binary search, lookup and other operations can use the principle of binary search. They allow fast lookup, addition and removal of items, and can be used to implement either dynamic sets of items, or lookup tables that allow finding an item by its key. 
The insert interface is defined as `void insert(Tree *tree, int value)`, which we insert an int value into the tree. However, a more portable interface is `void insert(Tree *tree, void *ptr)` which allowing to store a pointer value in the tree. The pointer can point to any other value/struct.
In order to call the insert function recursivily we have a workhorse function `insert_internal` to do most of the inserting job. Since we have to alter the `tree->root` pointer when new node is added into the tree, the working horse function's defination is `static void insert_internal(Node **node, int value)` in which the address of `tree->root` is passed as the first argument.
This part of code is below. When inserting a value, if the current position is pointing to NULL we have to allcate a Node structure and store value in it. Else we compare the value against the current node and insert the smaller value into the left part of the tree and the bigger value into the right part of the tree.
```c
void insert(Tree *tree, int value)
{
  insert_internal(&(tree->root),value);
}

static void insert_internal(Node **node, int value)
{
    if(*node == NULL)
    {
        Node * new_node = (Node *)malloc(sizeof(*node));
        new_node->left = NULL;
        new_node->right = NULL;
        new_node->value = value;
        *node = new_node;
    }
    else if (value > (*node)->value)
      insert_internal(&((*node)->right), value);
    else
      insert_internal(&((*node)->left), value);
}
```



Preorder traversing
--------
Pre-order traversing, also known as root-first traversing(which is more clear), is done by visiting the root first, then the left child, the the right child.
### by using recursive
Recursive make the preorder traversing much simpler and more clear to implement. Much the same as inserting we have two separte functions to do the job. The latter one `print_internal` is mainly used for recurse purposes.
```c
void   print(Tree *tree)
{
    if(tree->root == NULL)
    {
        printf("(null)");
        return;
    }

    printf("root first:\t");
    print_internal(tree->root);
} 
static void print_internal(Node *node)
{
    if(node!=NULL)
        printf("%d\t", node->value);

    if(node->left!=NULL)
        print_internal(node->left);
    if(node->right!=NULL)
        print_internal(node->right);
}
```
### by using stack
If we are not using recursive ways, we can also use stack to do the same. This saves lots of function calls which is likely to be much more faster. 
```c
void   print5(Tree *tree)
{
    Stack *stack;

    if(tree->root == NULL)
    {
        printf("(null)");
        return;
    }

    stack = new_stack();

    printf("root first:\t");
    print_internal5(stack,tree->root);
} 
static void print_internal5(Stack *stack, Node *node)
{

    if(node!=NULL)
        push(stack,(void *)node);

    while(!is_stack_empty(stack))
    {
        Node *top;

        top = (Node *)pop(stack);
        printf("%d\t",top->value);

        if(top->right!=NULL)
          push(stack,(void *)top->right);
        if(top->left!=NULL)
          push(stack,(void *)top->left);
    }
}
```
It is worth noticing, we have to push right part of the tree first and left part of the tree last, since the stack is LIFO data structure. So the left part is poped out first in the next loop. When the left part is poped out, its value is printed and the right/left part is pushed on the top of the stack. So, always the root first then the left part then the right part.

inorder traversing
--------
The inorder traversing is done by visiting the left child first, then the root, then the right child last.
```c
void   print2(Tree *tree)
{

    if(tree->root == NULL)
    {
        printf("(null)");
        return;
    }

    printf("root second:\t");
    print_internal2(tree->root);
} 
static void print_internal2(Node *node)
{
    if(node->left!=NULL)
        print_internal2(node->left);
    if(node!=NULL)
        printf("%d\t", node->value);
    if(node->right!=NULL)
        print_internal2(node->right);
}
```

postorder traversing
---------
```c
void   print3(Tree *tree)
{
    if(tree->root == NULL)
    {
        printf("(null)");
        return;
    }

    printf("root last:\t");
    print_internal3(tree->root);
} 

static void print_internal3(Node *node)
{

    if(node->left!=NULL)
        print_internal3(node->left);
    if(node->right!=NULL)
        print_internal3(node->right);
    if(node!=NULL)
        printf("%d\t", node->value);
}
```

level order traversing
----------
A queue(FIFO) can be used to do the level traversing of a tree. By put the root in queue, then the left child of the root in queue, then the right child of the tree in queue.
```c
void   print4(Tree *tree)
{
    Queue *queue;

    if(tree->root == NULL)
    {
        printf("(null)");
        return;
    }

    queue = new_queue();

    printf("level out:\t");
    print_internal4(queue,tree->root);
}
static void print_internal4(Queue * queue,Node *node)
{

    if(node!=NULL)
        inqueue(queue,(void *)node);

    while(!is_queue_empty(queue))
    {
        Node *front;

        front = (Node *)dequeue(queue);
        printf("%d\t",front->value);

        if(front->left!=NULL)
          inqueue(queue,(void *)front->left);
        if(front->right!=NULL)
          inqueue(queue,(void *)front->right);
    }
}

```
This algorithm is worth much thinking to have a full understanding of how it works.




Graphic output
----------
Any of the above traversing algorithm can be modified to print the edge between the parent and child. Making it much easy to adapt to the `graphviz` grammer. 
One sample of the Fig1 converted to graphviz format is following. Each line representing either a vertex of the tree or the edge of the tree. 
```
digraph G {
5
5 -> 3
3
3 -> 2
2
2 -> 1
1
3 -> 4
4
5 -> 6
6
6 -> 7
7
7 -> 8
8
}
```
It is done by the following code:
```c
void   print_graph(Tree *tree)
{
    if(tree->root == NULL)
    {
        printf("(null)");
        return;
    }

    printf("digraph G {\n");
    print_graph_internal(tree->root);
    printf("}\n");
}
static void print_graph_internal(Node *node)
{
    if(node!=NULL)
        printf("%d\n", node->value);

    if(node->left!=NULL)
    {
        printf("%d -> %d\n", node->value, node->left->value);
        print_graph_internal(node->left);
    }
    if(node->right!=NULL)
    {
        printf("%d -> %d\n", node->value, node->right->value);
        print_graph_internal(node->right);
    }
}

```
Later, issue command `dot -Tjpg file.dot -o file.jpg` to draw the actual graph as shown in figure1.

  [1]: http://static.zybuluo.com/shenyuflying/d5btsnjedki9jmw8klc8p6it/image_1bc2d7db4nop98t1ukvunk61p9.png