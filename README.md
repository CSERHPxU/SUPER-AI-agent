#include<iostream>
#include<vector>
#include<queue>
#include<climits>

using namespace std;
// we have defined the necessary header files here for this problem.
// If additional header files are needed in your program, please import here.

struct Node{
    long long val;
    Node * left;
    Node * right;
    Node(long long v) : val(v),left(nullptr),right(nullptr){}
};

Node* build(const vector<long long>& a, int& idx,long long low,long long high){
    if(idx < 0) return nullptr;
    
    long long v = a[idx];
    if(v <= low || v >= high) return nullptr;
    
    Node* root = new Node(v);
    idx--;
    root->right = build(a,idx,v,high);
    root->left = build(a,idx,low,v);
    
    return root;
}

long long getSum(Node* root){
    if(!root) return 0;
    queue<Node*> q;
    q.push(root);
    long long sum = 0;
    while(!q.empty()){
        int sz = q.size();
        long long rightmost = 0;
        for(int i = 0;i < sz ;i++){
            Node* cur =q.front();
            q.pop();
            
            rightmost = cur->val;
            if(cur->left) q.push(cur->left);
            if(cur->right) q.push(cur->right);
        }
        sum += rightmost;
    }
    return sum;
}

void destroy(Node* root){
    if(!root) return;
    destroy(root->left);
    destroy(root->right);
    delete root;
}

int main()
{
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    
    vector<long long> a;
    long long x;
    while(cin>>x){
        a.push_back(x);
    }
    int idx = static_cast<int>(a.size()) -1;
    
    Node* root = build(a,idx,LLONG_MIN,LLONG_MAX);
    if(idx != -1){
        cout << -1 << endl;
        destroy(root);
        return 0;
    }
    
    cout << getSum(root) << endl;
    destroy(root);
    return 0;
}
