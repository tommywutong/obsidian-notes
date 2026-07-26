---
title: "Leetcode hot 100 个人总结（持续更新）"
slug: "leetcode-hot-100-notes"
published: 2025-05-05
description: "笨人算法fresh man，如有纰漏万望您拨冗指正，不胜感激 二分查找 个人认为二分查找的难点主要分布在两点： 1. 不知道应该使用二分查找 2. 二分查找边界条件判断（ & nums, int target) { auto ans = lower bound(nums.begi"
tags: ["算法", "LeetCode"]
category: "工程"
draft: false
---

## 笨人算法fresh man，如有纰漏万望您拨冗指正，不胜感激


## 二分查找


个人认为二分查找的难点主要分布在两点：


1. 不知道应该使用二分查找


2. 二分查找边界条件判断（ <= or < ;  mid = left + 1 or left） 


###


### 35. 搜索插入位置


不做赘述，有问题的需巩固基础 


```cpp
class Solution {
public:
    int searchInsert(vector<int>& nums, int target) {
        auto ans = lower_bound(nums.begin(), nums.end(), target);
        int answer = ans - nums.begin();
        return answer;
    }
};
### 34. 在排序数组中查找元素的第一个和最后一个位置


本题可以算是lower_bound和upper_bound的模板题，但需注意：


upper_bound 是查找第一个大于target的位置


lower_bound 是查找第一个大于等于target的位置


两者均返回迭代器，所以需用auto。


```cpp
#include <algorithm>
class Solution {
public:
    vector<int> searchRange(vector<int>& nums, int target) {
        // if (!binary_search(nums.begin(), nums.end(), target))  {
        //     return {-1, -1};
        // }
        if (find(nums.begin(), nums.end(), target) == nums.end()) {
            return {-1, -1};
        }
        auto first = lower_bound(nums.begin(), nums.end(), target);
        int ans1 = first - nums.begin();
        auto last = upper_bound(nums.begin(), nums.end(), target);
        int ans2 = last - nums.begin() - 1;
        return {ans1, ans2};
    }
};
###


### 74. 搜索二维矩阵


本题可将二维矩阵视为一维排序数组，进而进行常规二分查找，公式题


```cpp
class Solution {
public:
    bool searchMatrix(vector<vector<int>>& matrix, int target) {
        if (matrix.empty() || matrix[0].empty()) {
            return false;
        }
        int m = matrix.size();
        int n = matrix[0].size();
        int l = 0, r = m * n - 1;
        while (l <= r) {
            int mid = l + (r - l) / 2;
            int col = mid % n;
            int row = mid / n;
            int midValue = matrix[row][col];

            if (midValue == target) {
                return true;
            } else if (midValue < target) {
                l = mid + 1;
            } else {
                r = mid - 1;
            }
        }

        return false;
    }
};
###


### 33. 搜索旋转排序数组


```cpp
class Solution {
public:
    int search(vector<int>& nums, int target) {
        int left = 0, right = nums.size() - 1;

        while (left <= right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] == target) {
                return mid;
            }

            // 左半部分有序
            if (nums[left] <= nums[mid]) {
                if (nums[left] <= target && target < nums[mid]) {
                    right = mid - 1;
                } else {
                    left = mid + 1;
                }
            }
            // 右半部分有序
            else {
                if (nums[mid] < target && target <= nums[right]) {
                    left = mid + 1;
                } else {
                    right = mid - 1;
                }
            }
        }

        return -1;
    }
};
###


### 153. 寻找旋转排序数组中的最小值


```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int n = nums.size();
        int l = 0, r = n - 1;
        int res = nums[0];
        while (l <= r) {
            int mid = l + (r - l) / 2;
            res = min(res, nums[mid]);
            if (nums[mid] >= nums[r]) {
                l = mid + 1;
            } else {
                r = mid - 1;
            }
        }
        return res;
    }
};
###


### 154. 寻找旋转排序数组中的最小值 II


 本题相比153，数组可能包含重复元素


Leetcode 153 假设 **没有重复元素**，所以 nums[mid] 和 nums[right] 总是能比较出“最小值在哪边”。


但是154的问题是：nums[mid] == nums[right] 这时无法判断哪边有最小值，要特殊处理。


因此，需要在程序中特殊处理这种情况


```cpp
class Solution {
public:
    int findMin(vector<int>& nums) {
        int left = 0, right = nums.size() - 1;

        while (left < right) {
            int mid = left + (right - left) / 2;

            if (nums[mid] < nums[right]) {
                // 最小值在左边（包含 mid）
                right = mid;
            } else if (nums[mid] > nums[right]) {
                // 最小值在右边（不包含 mid）
                left = mid + 1;
            } else {
                // nums[mid] == nums[right]，无法判断最小值在哪边
                // 所以只能保守地缩小边界
                right--; // ✅ 注意不能直接排除 mid
            }
        }

        return nums[left];  // 此时 left == right
    }
};
###


### 4. 寻找两个正序数组的中位数


```cpp
class Solution {
public:
    double findMedianSortedArrays(vector<int>& nums1, vector<int>& nums2) {
        int m = nums1.size(), n = nums2.size();
        vector<int> a(m + n);
        int i = 0, j = 0, k = 0;
        while (i < m && j < n) {
            if (nums1[i] < nums2[j])
                a[k++] = nums1[i++];
            else
                a[k++] = nums2[j++];
        }

        while (i < m) {
            a[k++] = nums1[i++];
        }

        while (j < n) {
            a[k++] = nums2[j++];
        }

        int len = m + n;
        if (len % 2 == 0) {
            return (a[len / 2] + a[len / 2 - 1]) / 2.0;
        } else {
            return a[len / 2];
        }
    }
};
```


## 链表


### 160. 相交链表


**逻辑讲解：**


- temp1 从 headA 开始，temp2 从 headB 开始。
- 每次都向下走一个节点，如果走到末尾就跳到另一个链表的头。
- 如果两个链表有交点，他们最终会在某一个相交节点相遇（地址相同）。
- 如果没有交点，他们都走到 nullptr，循环结束，返回空。

**为什么这样能找到相交点？**


因为两个指针在**第二次遍历**时，走的长度是一样的：


- 第一次遍历：temp1 走了 A 的长度，接着走 B；
- temp2 走了 B 的长度，接着走 A；
- 所以无论两个链表的长度如何，最终走的路径长度是一样的（A + B）。

当相交时，他们在同一个节点相遇；不相交就都走到末尾 nullptr，也会相等。


```cpp
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
    public:
        ListNode *getIntersectionNode(ListNode *headA, ListNode *headB) {
            ListNode* temp1 = headA;
            ListNode* temp2 = headB;
            while(temp1 != temp2) {
                temp1 = temp1 == nullptr ? headB : temp1->next;
                temp2 = temp2 == nullptr ? headA : temp2->next;
            }
            return temp1;
        }
    };
###


### 206. 反转链表


1. 迭代法


```cpp
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* curr = head;
    while (curr) {
        ListNode* next = curr->next; // 暂存下一节点
        curr->next = prev;           // 当前节点指向前一个
        prev = curr;                 // prev 向后移动
        curr = next;                 // curr 向后移动
    }
    return prev;
}
```


2. 递归法


```cpp
ListNode* reverseList(ListNode* head) {
    if (head == nullptr || head->next == nullptr) return head;
    ListNode* newHead = reverseList(head->next);
    head->next->next = head; // 把当前节点挂到后面那个节点的后面
    head->next = nullptr;    // 当前节点断开原来的 next
    return newHead;
}
###


### 234. 回文链表


我们可以使用快慢指针定位到链表中间位置，反转后半段链表，然后比较前后半段链表


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
        bool isPalindrome(ListNode* head) {
            ListNode* fast = head;
            ListNode* slow = head;
            while(fast && fast->next){
                ListNode* temp = fast -> next;
                fast = temp -> next;
                slow = slow -> next;
            }
            //反转后半部分
            ListNode* prev = nullptr;
            ListNode* cur = slow;
            while(cur){
                ListNode*next = cur->next;
                cur -> next = prev;
                prev = cur;
                cur = next;
            }
            //比较
            ListNode* temp1 = head;
            ListNode* temp2 = prev;
            while(temp2){
                if(temp1->val!=temp2->val){
                    return false;
                }
                temp1 = temp1->next;
                temp2 = temp2->next;
            }
            return true;
        }
    };
###


### 142. 环形链表 II


依旧是快慢指针，如果快慢指针重合，则说明链表为环形


两个指针相遇以后，其中一个指针从头开始，另一个从相遇点出发。两者每次都走一步，再次相遇点节点就是环形入口。


** 数学证明（简化版）：**


设：


- 头到入环点的距离为 a
- 入环点到相遇点的距离为 b
- 环的长度为 c

相遇时：


- 慢指针走了 a + b 步；
- 快指针走了 a + b + n*c 步（多走了整圈）

由于快指针速度是慢指针的两倍：


2(a + b) = a + b + n*c


化简得：a = n*c - b


这意味着从头到入环点的距离 a，和从相遇点再走 c - b 步，会在入环点相遇。


```cpp
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        ListNode* fast = head;
        ListNode* slow = head;
        while(fast && fast -> next){
            fast = fast -> next -> next;
            slow = slow -> next;
            if(fast == slow){
                ListNode* temp1 = head;
                ListNode* temp2 = slow;
                while(temp1 != temp2){
                    temp1 = temp1->next;
                    temp2 = temp2-> next;
                }
                return temp1;
            }
        }
        return NULL;
    }
};
###


### 2. 两数相加


类似于高精度加法，只是由数组变换成链表，模拟加法的竖式运算过程


需注意处理进位


```cpp
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        ListNode* dummy = new ListNode(0);
        ListNode* cur = dummy;
        int carry = 0;
        while (l1 || l2 || carry) {
            int sum = carry;
            if (l1) {
                sum += l1->val;
                l1 = l1->next;
            }
            if (l2) {
                sum += l2->val;
                l2 = l2->next;
            }
            carry = sum / 10;
            cur->next = new ListNode(sum % 10);
            cur = cur->next;
        }
        return dummy->next;
    }
};
### 19. 删除链表的倒数第 N 个结点


很典型的双指针快慢指针类型，快指针先走n，随后快慢同步走，慢指针停留在倒数第n个节点


```cpp
class Solution {
public:
    ListNode* removeNthFromEnd(ListNode* head, int n) {
        ListNode* dummy = new ListNode(0);
        dummy -> next = head;
        ListNode* fast = dummy;
        ListNode* slow = dummy;
        for(int i = 0; i <= n ;i++){
            fast  = fast -> next;
        }
        while(fast){
            fast = fast-> next;
            slow = slow -> next;
        }
        ListNode* nxt = slow -> next;
        slow -> next = nxt ->next;
        delete nxt;
        ListNode* res = dummy -> next;
        delete dummy;
        return res;
    }
};
### 24. 两两交换链表中的节点


```cpp
class Solution {
public:
    ListNode* swapPairs(ListNode* head) {
        ListNode* dummy = new ListNode(0);
        dummy -> next = head;
        ListNode* cur = dummy;
        while(cur -> next && cur -> next -> next){
            ListNode* temp1 = cur -> next;
            ListNode* temp2 = cur -> next -> next -> next;
            cur -> next = cur -> next -> next;
            cur -> next -> next = temp1;
            cur -> next -> next -> next = temp2;
            cur = cur -> next -> next;
        }
        ListNode* result = dummy -> next;
        delete dummy;
        return result;
    }
};
### 25. K 个一组翻转链表


其实就是上一道题的升级版。 


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
            //计算链表长度
            for(ListNode* cur = head;cur;cur = cur -> next){
                n++;
            }
            ListNode dummy (0,head);
            ListNode* p0 = &dummy;
            ListNode* prev = nullptr;
            ListNode* cur = head;
            //分成k个一组
            for(; n >= k;n -= k){
                for(int i = 0; i < k;i++){
                    ListNode* next = cur -> next;
                    cur -> next = prev;
                    prev = cur;
                    cur = next;
                }
                // 把翻转后的链表接回前后内容
                ListNode* nxt = p0 -> next;
                p0->next->next = cur;
                p0 -> next = prev;
                p0 = nxt;
            }
            return dummy.next;
    }
};
### 138. 随机链表的复制


方法1 ：第一次遍历，复制每个节点，并建立原节点到新节点映射 old -> new; 第二次遍历，链接新节点到next和random。返回新链表的头节点。


```cpp
class Solution {
public:
    Node* copyRandomList(Node* head) {
        if (!head) return nullptr;

        unordered_map<Node*, Node*> oldToNew;

        // 第一步：复制每个节点
        for (Node* cur = head; cur; cur = cur->next) {
            oldToNew[cur] = new Node(cur->val);
        }

        // 第二步：连边
        for (Node* cur = head; cur; cur = cur->next) {
            oldToNew[cur]->next = oldToNew[cur->next];
            oldToNew[cur]->random = oldToNew[cur->random];
        }

        return oldToNew[head];
    }
};
```


方法2：在每个节点后面插入一个副本，比如 A → A’ → B → B’ …，第二遍设置副本的random，比如   A’->random = A->random->next；最后拆分链表，把原始链表和副本链表分开


```cpp
class Solution {
public:
    Node* copyRandomList(Node* head) {
        if (!head) return nullptr;

        // 第一步：在每个节点后插入复制节点
        for (Node* cur = head; cur;) {
            Node* copy = new Node(cur->val);
            copy->next = cur->next;
            cur->next = copy;
            cur = copy->next;
        }

        // 第二步：复制 random 指针
        for (Node* cur = head; cur; cur = cur->next->next) {
            if (cur->random) {
                cur->next->random = cur->random->next;
            }
        }

        // 第三步：拆分两个链表
        Node* dummy = new Node(0);
        Node* copyCur = dummy;
        for (Node* cur = head; cur;) {
            Node* copy = cur->next;
            copyCur->next = copy;
            copyCur = copy;

            cur->next = copy->next;
            cur = cur->next;
        }

        return dummy->next;
    }
};
```


很多同学可能会疑惑：一道链表题怎么要用到哈希表呢？


本题的特点在于：普通的复制链表，只要复制next就可以了，而这道题中有random。你不能一边复制一遍正确指向random，因为你不知道random指向的新节点在哪


哈希表的作用： 记录原节点和新节点的对应关系。当遇到某个random指向的原节点时，可以马上找到对应新节点。是不是非常美妙？ 


### 148. 排序链表


本题可以借用归并排序的思想，然后新建一个合并链表函数。


```cpp
class Solution {
public:
    ListNode* sortList(ListNode* head) {
        if (!head || !head->next) return head;

        // 找中点
        ListNode* slow = head, *fast = head->next;
        while (fast && fast->next) {
            slow = slow->next;
            fast = fast->next->next;
        }

        // 分割
        ListNode* mid = slow->next;
        slow->next = nullptr;

        // 递归排序左右部分
        ListNode* left = sortList(head);
        ListNode* right = sortList(mid);

        // 合并
        return merge(left, right);
    }

    ListNode* merge(ListNode* l1, ListNode* l2) {
        ListNode dummy(0), *tail = &dummy;
        while (l1 && l2) {
            if (l1->val < l2->val) {
                tail->next = l1;
                l1 = l1->next;
            } else {
                tail->next = l2;
                l2 = l2->next;
            }
            tail = tail->next;
        }
        tail->next = l1 ? l1 : l2;
        return dummy.next;
    }
};
### 23. 合并 K 个升序链表


本题虽然标记为困难，但是可以不断调用合并两个升序链表暴力合并k个升序链表


```cpp
class Solution {
public:
    ListNode* mergetwoLists(ListNode* list1, ListNode* list2) {
        ListNode* dummy = new ListNode(0);
        ListNode* tail = dummy;
        while (list1 && list2) {
            if (list1->val < list2->val) {
                tail->next = list1;
                list1 = list1->next;
            } else {
                tail->next = list2;
                list2 = list2->next;
            }
            tail = tail->next;
        }
        if (list1) {
            tail->next = list1;
        } else {
            tail->next = list2;
        }
        return dummy->next;
    }
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        if (lists.empty()) {
            return nullptr;
        }
        ListNode* dummy = new ListNode(0);
        int n = lists.size();
        for (int i = 0; i < n; i++) {
            if (lists[i]) {
                dummy->next = mergetwoLists(dummy->next, lists[i]);
            }
        }
        return dummy->next;
    }
};
```


## 普通数组


### 53. 最大子数组和


```cpp
class Solution {
public:
    int maxSubArray(vector<int>& nums) {
        int n = nums.size();
        int ans = nums[0];
        int currentSum = nums[0];
        for (int i = 1; i < n; i++) {
            currentSum = max(nums[i], currentSum + nums[i]);
            ans = max(ans, currentSum);
        }
        return ans;
    }
};
### 56. 合并区间


先按照左端点排序，用一个结果数组result来合并区间。如果当前区间跟result的最后一个区间没有重叠，直接添加；如果有重叠，修改result最后一个区间的右端点


```cpp
class Solution {
public:
    vector<vector<int>> merge(vector<vector<int>>& intervals) {
        if (intervals.size() == 0) {
            return {};
        }
        sort(intervals.begin(), intervals.end());
        vector<vector<int>> merged;
        for (int i = 0; i < intervals.size(); ++i) {
            int L = intervals[i][0], R = intervals[i][1];
            if (!merged.size() || merged.back()[1] < L) {
                merged.push_back({L, R});
            }
            else {
                merged.back()[1] = max(merged.back()[1], R);
            }
        }
        return merged;
    }
};
### 189. 轮转数组


可以自己手动模拟一下


```cpp
class Solution {
public:
    void rotate(vector<int>& nums, int k) {
        int n = nums.size();
        k = k % n;
        reverse(nums.begin(), nums.end());          //整体翻转
        reverse(nums.begin(), nums.begin() + k);    //翻转前k个
        reverse(nums.begin() + k, nums.end());      //翻转后n-k个
    }
};
### 238. 除自身以外数组的乘积


```cpp
class Solution {
public:
    vector<int> productExceptSelf(vector<int>& nums) {
        int n = nums.size();
        vector<int> answer(n, 1);

        // 先计算左边所有乘积
        for (int i = 1; i < n; ++i) {
            answer[i] = answer[i - 1] * nums[i - 1];
        }

        // 再从右往左乘
        int right = 1;
        for (int i = n - 1; i >= 0; --i) {
            answer[i] *= right;
            right *= nums[i];
        }

        return answer;
    }
};
### 41. 缺失的第一个正数


本题核心目标是让第 i 位上的数字是 i + 1。


** 完整步骤**


- 遍历整个数组： 如果 nums[i] 是合法数字（1 ~ n）并且没在正确位置上，
- 就交换 nums[i] 和 nums[nums[i] - 1]。
- 注意是**不断交换直到当前位置合法**！

- 再次遍历数组： 找第一个 nums[i] != i+1 的位置，
- 答案就是 i+1。

- 如果遍历完都符合，就返回 n+1。


```cpp
class Solution {
public:
    int firstMissingPositive(vector<int>& nums) {
        int n = nums.size();

        for (int i = 0; i < n; ++i) {
            while (nums[i] > 0 && nums[i] <= n && nums[nums[i]-1] != nums[i]) {
                swap(nums[i], nums[nums[i] - 1]);
            }
        }

        for (int i = 0; i < n; ++i) {
            if (nums[i] != i + 1) {
                return i + 1;
            }
        }

        return n + 1;
    }
};
```


## 动态规划


### 322. 零钱兑换


很常规的完全背包dp，需要注意： 完全背包中内循环外循环均为正序遍历


```cpp
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        const int INF = amount + 1;
        vector<int> dp(amount + 1, INF);
        dp[0] = 0;

        for (int coin : coins) {
            for (int i = coin; i <= amount; ++i) {
                dp[i] = min(dp[i], dp[i - coin] + 1);
            }
        }

        return dp[amount] == INF ? -1 : dp[amount];
    }
};
### 213. 打家劫舍 II


hot100中本来为打家劫舍1，但是2中也有1的模板且多了一层思维，故替换为2.


因本题中房屋为环形，为最大程度利用1中的代码，我们可以分成两种情况：


1. 不偷第一个


2. 不偷最后一个


每种情况都可以应用打家劫舍1的代码，最后讲两种情况取一个最大值即可


```cpp
class Solution {
public:
    int robb(vector<int>& nums, int i, int j) {
        int n = j - i + 1;
        if (n == 1) {
            return nums[i];
        }
        vector<int> dp(n, 0);
        dp[0] = nums[i];
        dp[1] = max(nums[i], nums[i + 1]);
        for (int k = 2; k < n; k++) {
            dp[k] = max(dp[k - 1], dp[k - 2] + nums[i + k]);
        }
        return dp[n - 1];
    }
    int rob(vector<int>& nums) {
        int n = nums.size();
        if (n == 0) return 0;
        if (n == 1) return nums[0];
        //不偷第一个
        int case1 = robb(nums, 1, n - 1);
        //不偷最后一个
        int case2 = robb(nums, 0, n - 2);
        return max(case1, case2);
    }
};
```


需要注意：robb函数在添加左右范围后dp数组的初始值和递归公式都会有细微的变化（以前基于0，现在基于i）因此一个重新整理思路而非盲目默写﫡


### 279. 完全平方数


需要一些小小的应变，主要还是考察对动态规划双层遍历的理解


```cpp
class Solution {
public:
    int numSquares(int n) {
        vector<int> dp(n + 1, INT_MAX);
        dp[0] = 0;

        for (int i = 1; i <= n; ++i) {
            for (int j = 1; j * j <= i; ++j) {
                dp[i] = min(dp[i], dp[i - j * j] + 1);
            }
        }

        return dp[n];
    }
};
### 322. 零钱兑换


基本还是不需要变通，乱套公式即可得分


```cpp
class Solution {
public:
    int coinChange(vector<int>& coins, int amount) {
        const int INF = amount + 1;
        vector<int> dp(amount + 1, INF);
        dp[0] = 0;

        for (int coin : coins) {
            for (int i = coin; i <= amount; ++i) {
                dp[i] = min(dp[i], dp[i - coin] + 1);
            }
        }

        return dp[amount] == INF ? -1 : dp[amount];
    }
};
### 139. 单词拆分


我们定义一个bool数组dp，dp[i] = true / false 表示前i个字符能不能被拆分


```cpp
class Solution {
public:
    bool wordBreak(string s, vector<string>& wordDict) {
        unordered_set<string> dict(wordDict.begin(), wordDict.end());
        int n = s.size();
        vector<bool> dp(n + 1, false);
        dp[0] = true; // 空串可以拆分

        for (int i = 1; i <= n; ++i) {
            for (int j = 0; j < i; ++j) {
                if (dp[j] && dict.count(s.substr(j, i - j))) {
                    dp[i] = true;
                    break; // 找到一种拆分方式就可以停止了
                }
            }
        }

        return dp[n];
    }
};
```


## 栈


栈常用于具有后进先出特性的题目，常用于匹配或寻找临近元素


### 20. 有效的括号


 一道很经典的栈匹配算法


```cpp
class Solution {
public:
    bool isValid(string s) {
        int n = s.size();
        if (s.size() % 2 != 0) {
            return false;
        }
        stack<char> stk;
        for (char c : s) {
            if (c == '(') {
                stk.push(')');
            } else if (c == '[') {
                stk.push(']');
            } else if (c == '{') {
                stk.push('}');
            } else if (stk.empty() || stk.top() != c) {
                return false;
            } else {
                stk.pop();
            }
        }
        return stk.empty();
    }
};
### 394. 字符串解码


本题特点在于字符串括号内重复嵌套括号，因此需要用到栈的知识来匹配。使用辅助栈来解题，需注意，如果录入的是数字，需将数字前一位 *10 


```cpp
class Solution {
public:
    string decodeString(string s) {
    stack<int> countStack;
    stack<string> stringStack;
    string currentString = "";
    int currentNum = 0;

    for (char ch : s) {
        if (isdigit(ch)) {
            currentNum = currentNum * 10 + (ch - '0');

        } else if (ch == '[') {
            countStack.push(currentNum);
            stringStack.push(currentString);
            currentString = "";
            currentNum = 0;
        } else if (ch == ']') {
            string prevString = stringStack.top(); stringStack.pop();
            int repeatTimes = countStack.top(); countStack.pop();
            string temp = "";
            for (int i = 0; i < repeatTimes; ++i) {
                temp += currentString;
            }
            currentString = prevString + temp;
        } else {
            currentString += ch;
        }
    }
    return currentString;
    }
};
### 739. 每日温度


本题为单调栈的经典例题。维护一个存储下标的单调栈，从栈底到栈顶的下标对应的温度列表中的温度依次递减。如果一个下标在单调栈里，则表示尚未找到下一次温度更高的下标。


正向遍历温度列表。对于温度列表中的每个元素 temperatures[i]，如果栈为空，则直接将 i 进栈，如果栈不为空，则比较栈顶元素 prevIndex 对应的温度 temperatures[prevIndex] 和当前温度 temperatures[i]，如果 temperatures[i] > temperatures[prevIndex]，则将 prevIndex 移除，并将 prevIndex 对应的等待天数赋为 i - prevIndex，重复上述操作直到栈为空或者栈顶元素对应的温度小于等于当前温度，然后将 i 进栈。


为什么可以在弹栈的时候更新 ans[prevIndex] 呢？因为在这种情况下，即将进栈的 i 对应的 temperatures[i] 一定是 temperatures[prevIndex] 右边第一个比它大的元素，试想如果 prevIndex 和 i 有比它大的元素，假设下标为 j，那么 prevIndex 一定会在下标 j 的那一轮被弹掉。


由于单调栈满足从栈底到栈顶元素对应的温度递减，因此每次有元素进栈时，会将温度更低的元素全部移除，并更新出栈元素对应的等待天数，这样可以确保等待天数一定是最小的。


以下用一个具体的例子帮助读者理解单调栈。对于温度列表 [73,74,75,71,69,72,76,73]，单调栈 stack 的初始状态为空，答案 ans 的初始状态是 [0,0,0,0,0,0,0,0]，按照以下步骤更新单调栈和答案，其中单调栈内的元素都是下标，括号内的数字表示下标在温度列表中对应的温度。


```cpp
class Solution {
public:
    vector<int> dailyTemperatures(vector<int>& temperatures) {
        int n = temperatures.size();
        vector<int> ans(n);
        stack<int> s;
        for (int i = 0; i < n; ++i) {
            while (!s.empty() && temperatures[i] > temperatures[s.top()]) {
                int previousIndex = s.top();
                ans[previousIndex] = i - previousIndex;
                s.pop();
            }
            s.push(i);
        }
        return ans;
    }
};
```


 


## 矩阵


矩阵系列问题更多的是数组和模拟思路的结合，主要考察代码能力而非算法思想


### 73. 矩阵置零


```cpp
class Solution {
public:
    void setZeroes(vector<vector<int>>& matrix) {
        int m = matrix.size();
        int n = matrix[0].size();
        vector<int> row(m), col(n);
        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                if (!matrix[i][j]) {
                    row[i] = col[j] = true;
                }
            }
        }
        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) {
                if (row[i] || col[j]) {
                    matrix[i][j] = 0;
                }
            }
        }
    }
};
### 54. 螺旋矩阵


一道非常考验代码能力的题，层层循环的嵌套控制边界。在写代码前整理好思路 


```cpp
class Solution {
public:
    vector<int> spiralOrder(vector<vector<int>>& matrix) {
        vector<int> result;
        if (matrix.empty() || matrix[0].empty())
            return result;

        int m = matrix.size();    // 行数
        int n = matrix[0].size(); // 列数

        int top = 0, bottom = m - 1;
        int left = 0, right = n - 1;

        while (top <= bottom && left <= right) {
            // 从左到右
            for (int j = left; j <= right; ++j) {
                result.push_back(matrix[top][j]);
            }
            top++;

            // 从上到下
            for (int i = top; i <= bottom; ++i) {
                result.push_back(matrix[i][right]);
            }
            right--;

            // 从右到左（注意是否还存在这一行）
            if (top <= bottom) {
                for (int j = right; j >= left; --j) {
                    result.push_back(matrix[bottom][j]);
                }
                bottom--;
            }

            // 从下到上（注意是否还存在这一列）
            if (left <= right) {
                for (int i = bottom; i >= top; --i) {
                    result.push_back(matrix[i][left]);
                }
                left++;
            }
        }

        return result;
    }
};
### 48. 旋转图像


```cpp
class Solution {
public:
    void rotate(vector<vector<int>>& matrix) {
        int n = matrix.size();
        // 深拷贝 matrix -> tmp
        vector<vector<int>> tmp = matrix;
        // 根据元素旋转公式，遍历修改原矩阵 matrix 的各元素
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                matrix[j][n - 1 - i] = tmp[i][j];
            }
        }
    }
};
```

---

原文发布于 CSDN：[Leetcode hot 100 个人总结（持续更新）](https://blog.csdn.net/2402_86720949/article/details/147483386)
