---
title: "Leetcode经典链表问题之反转链表"
slug: "reverse-linked-list"
published: 2025-03-22
description: "206. 反转链表 力扣（LeetCode） 92. 反转链表 II 力扣（LeetCode） 反转链表2，只反转部分链表 关于哨兵节点 哨兵节点的作用 ✅ (1) 确保 m 1 位置正确连接 • 反转部分链表时，m 1 位置的节点需要正确指向反转后的 m 位置。 • 哨兵节点"
tags: ["算法", "LeetCode", "链表"]
category: "工程"
draft: false
---

## 206. 反转链表 - 力扣（LeetCode）


```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverseList(ListNode* head) {
        ListNode* prev = nullptr;
        ListNode* next = nullptr;
        ListNode* temp = head;
        while(temp!= nullptr){
            next = temp -> next;
            temp -> next = prev;
            prev = temp;
            temp = next;
        }
        return prev;
    }
};
```


## 92. 反转链表 II - 力扣（LeetCode）


反转链表2，只反转部分链表


```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverseBetween(ListNode* head, int left, int right) {
        ListNode* dummy = new ListNode(0);
        dummy->next = head;
        ListNode* p0 = dummy;
        for(int i = 1;i < left;i++){    //i 从1开始，移动left- 1次，指向left前一个节点
            p0 = p0->next;
        }
        ListNode* prev = nullptr;   // 不是p0
        ListNode* cur = p0 -> next;
        for(int i = left ;i <= right;i++){
            ListNode* next = cur->next;
            cur -> next = prev;
            prev = cur;
            cur = next;
        }
        p0 -> next -> next = cur;
        p0 -> next = prev;
        return dummy -> next;
    }
};
### 关于哨兵节点


** 哨兵节点的作用**


**✅ (1) 确保 m-1 位置正确连接**


• 反转部分链表时，m-1 位置的节点需要正确指向反转后的 m 位置。


• 哨兵节点 dummy 让 prev 总是存在，无论 m=1 还是 m>1，都可以通过 dummy->next 访问 m-1 位置。


**✅ (2) 处理 m=1 的情况，防止 head 丢失**


• 反转部分链表时，如果 m=1，head 需要更新。


• 如果直接修改 head，可能导致 head 丢失，难以返回正确的新头。


• 使用 dummy，dummy->next 总是指向 head，即使 head 改变，我们仍然可以返回 dummy.next，保证正确性。


## 25. K 个一组翻转链表 - 力扣（LeetCode）


```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode() : val(0), next(nullptr) {}
 *     ListNode(int x) : val(x), next(nullptr) {}
 *     ListNode(int x, ListNode *next) : val(x), next(next) {}
 * };
 */
class Solution {
public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        int n = 0;
        ListNode* temp = head;
        while(temp){
            n++;
            temp = temp -> next;
        }
        ListNode dummy(0,head);
        ListNode* p0 = &dummy;
        ListNode* prev = nullptr;
        ListNode* cur = head;
        for(;n >= k; n -= k){
            for(int i = 0; i < k ;i++){
                ListNode* next= cur->next;
                cur -> next = prev;
                prev = cur;
                cur = next;
            }
            ListNode* nxt = p0->next;
            p0 -> next -> next = cur;
            p0 -> next = prev;
            p0 = nxt;
        }
        return dummy.next;
    }
};
```

---

原文发布于 CSDN：[Leetcode经典链表问题之反转链表](https://blog.csdn.net/2402_86720949/article/details/146438398)
