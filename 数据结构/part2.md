# 二叉树

二叉树 ```c++``` 实现

```BinaryTree```类定义：

```c++
/**
 * 二叉树结点定义
 * @param data 数据域
 * @param left 左孩子结点
 * @param right 右孩子结点
 */
template<class T>
struct TreeNode {
    T data;
    TreeNode<T> *left;
    TreeNode<T> *right;
};

template<class T>
class BinaryTree {
public:
    BinaryTree();

    void CreateTree();

    void Preorder();

    void InOrder();

    void PostOrder();

    int Size();

    //根结点
    TreeNode<T> *root;

private:
    int size;

    TreeNode<T> *InnerCreateTree(TreeNode<T> *root);

    void InnerPreorder(TreeNode<T> *root);

    void InnerInOrder(TreeNode<T> *root);

    void InnerPostOrder(TreeNode<T> *root);
};
```

前序生成二叉树：

```c++
/**
 * 前序生成二叉树
 */
template<class T>
void BinaryTree<T>::CreateTree() {
    root = InnerCreateTree(root);
}

template<class T>
TreeNode<T> *BinaryTree<T>::InnerCreateTree(TreeNode<T> *root) {
    char data;
    if ((data = getchar()) == '#') {
        root = nullptr;
    } else {
        root = new TreeNode<T>;
        root->data = data;
        cout << "add node:" << data << endl;
        size++;
        root->left = InnerCreateTree(root->left);
        root->right = InnerCreateTree(root->right);
    }
    return root;
}
```



前序遍历二叉树：

```c++
/**
 * 前序遍历二叉树
 */
template<class T>
void BinaryTree<T>::Preorder() {
    InnerPreorder(root);
}

template<class T>
void BinaryTree<T>::InnerPreorder(TreeNode<T> *root) {
    if (!root) {
        return;
    }
    cout << root->data << " ";
    InnerPreorder(root->left);
    InnerPreorder(root->right);
}
```



中序遍历二叉树：

```c++
/**
 * 中序遍历二叉树
 */
template<class T>
void BinaryTree<T>::InOrder() {
    InnerInOrder(root);
}

template<class T>
void BinaryTree<T>::InnerInOrder(TreeNode<T> *root) {
    if (!root) {
        return;
    }
    InnerPreorder(root->left);
    cout << root->data << " ";
    InnerPreorder(root->right);
}
```



后序遍历二叉树：

```c++
/**
 * 后序遍历二叉树
 */
template<class T>
void BinaryTree<T>::PostOrder() {
    InnerPostOrder(root);
}

template<class T>
void BinaryTree<T>::InnerPostOrder(TreeNode<T> *root) {
    if (!root) {
        return;
    }
    InnerPreorder(root->left);
    InnerPreorder(root->right);
    cout << root->data << " ";
}
```