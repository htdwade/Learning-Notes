# LeetCode n数之和系列问题

## 1. 两数之和

给定一个整数数组 nums 和一个目标值 target，请你在该数组中找出和为目标值的那 两个 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，你不能重复利用这个数组中同样的元素。

示例:

```java
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```

### 思路

使用哈希表。遍历数组 nums，i 为当前下标，每个值都判断map中是否存在 target-nums[i] 的 key 值，如果存在则找到了这两个值，如果不存在则将当前的 (nums[i],i) 存入 map 中，继续遍历直到找到为止。

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> hash = new HashMap<>();
        int[] res = new int[2];
        for(int i = 0; i < nums.length; i++){
            int numToFind = target - nums[i];
            if(hash.containsKey(numToFind)){
                res[0] = hash.get(numToFind);
                res[1] = i;
                break;
            }
            hash.put(nums[i], i);
        }
        return res;
    }
}
```

## 167. 两数之和 II - 输入有序数组

给定一个已按照升序排列 的有序数组，找到两个数使得它们相加之和等于目标数。

函数应该返回这两个下标值 index1 和 index2，其中 index1 必须小于 index2。

说明:

* 返回的下标值（index1 和 index2）不是从零开始的。
* 你可以假设每个输入只对应唯一的答案，而且你不可以重复使用相同的元素。

示例:
```java
输入: numbers = [2, 7, 11, 15], target = 9
输出: [1,2]
解释: 2 与 7 之和等于目标数 9 。因此 index1 = 1, index2 = 2 。
```

### 思路

使用双指针，一个指针指向值较小的元素，一个指针指向值较大的元素。指向较小元素的指针从头向尾遍历，指向较大元素的指针从尾向头遍历。

* 如果两个指针指向元素的和 sum == target，那么得到要求的结果；
* 如果 sum > target，移动较大的元素，使 sum 变小一些；
* 如果 sum < target，移动较小的元素，使 sum 变大一些。

```java
class Solution {
    public int[] twoSum(int[] numbers, int target) {
        int[] res = new int[2];
        int left = 0, right = numbers.length - 1;
        while(left < right) {
            int sum = numbers[left] + numbers[right];
            if(sum == target) {
                res[0] = left + 1;
                res[1] = right + 1;
                return res;
            }
            else if(sum < target)
                left++;
            else
                right--;
        }
        return res;
    }
}
```

## 15. 三数之和

给你一个包含 n 个整数的数组 nums，判断 nums 中是否存在三个元素 a，b，c ，使得 a + b + c = 0 ？请你找出所有满足条件且不重复的三元组。

注意：答案中不可以包含重复的三元组。

示例：
```java
给定数组 nums = [-1, 0, 1, 2, -1, -4]，

满足要求的三元组集合为：
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```

### 思路

* 如果数组为 null 或者数组长度小于 3，返回 []。
* 对数组进行排序。
* 遍历排序后数组，若 nums[i] > 0：因为已经排序好，所以后面不可能有三个数加和等于 0，直接返回结果。
* 对于重复元素：跳过，避免出现重复解
* 令左指针 L=i+1，右指针 R=n-1，当 L < R 时，执行循环：
    * 若 nums[i]+nums[L]+nums[R]==0，加入结果集，判断左界和右界是否和下一位置重复，去除重复解。并同时将 L,R 移到下一位置，寻找新的解；
    * 若和大于 0，说明 nums[R] 太大，R 左移；
    * 若和小于 0，说明 nums[L] 太小，L 右移。

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> res = new ArrayList<>();
        if(nums == null || nums.length < 3)
            return res;
        Arrays.sort(nums);
        int n = nums.length;
        for(int i = 0; i < n - 2; i++) {
            if(nums[i] > 0)
                return res;
            if(i > 0 && nums[i] == nums[i - 1])
                continue;
            int left = i + 1, right = n - 1;
            while(left < right) {
                int sum = nums[i] + nums[left] + nums[right];
                if(sum == 0) {
                    res.add(Arrays.asList(nums[i], nums[left], nums[right]));
                    while(left < right && nums[left] == nums[left + 1])
                        left++;
                    while(left < right && nums[right] == nums[right - 1])
                        right--;
                    left++;
                    right--;
                }
                else if(sum < 0)
                    left++;
                else
                    right--;
            }
        }
        return res;
    }
}
```

## 16. 最接近的三数之和

给定一个包括 n 个整数的数组 nums 和 一个目标值 target。找出 nums 中的三个整数，使得它们的和与 target 最接近。返回这三个数的和。假定每组输入只存在唯一答案。
```java
例如，给定数组 nums = [-1，2，1，-4], 和 target = 1.

与 target 最接近的三个数的和为 2. (-1 + 2 + 1 = 2).
```

### 思路

采用3sum的思路，先数组排序，遍历数组，固定nums[i], 令left=i+1，right=n-1。
* 若nums[i]+nums[left]+nums[right] == target，则直接返回和。
* 若和大于target，right--；
* 若和小于target，left++；
* 更新结果为距离target更近的和。

```java
class Solution {
    public int threeSumClosest(int[] nums, int target) {
        if(nums == null || nums.length < 3)
            return 0;
        Arrays.sort(nums);
        int res = nums[0] + nums[1] + nums[2];
        int n = nums.length;
        for(int i = 0; i < n - 2; i++) {
            int left = i + 1, right = n - 1;
            while(left < right) {
                int sum = nums[i] + nums[left] + nums[right];
                if(sum == target)
                    return sum;
                else if(sum < target)
                    left++;
                else
                    right--;
                if(Math.abs(sum - target) < Math.abs(res - target))
                    res = sum;
            }
        }
        return res;
    }
}
```

## 18. 四数之和

给定一个包含 n 个整数的数组 nums 和一个目标值 target，判断 nums 中是否存在四个元素 a，b，c 和 d ，使得 a + b + c + d 的值与 target 相等？找出所有满足条件且不重复的四元组。

注意：

答案中不可以包含重复的四元组。

示例：
```java
给定数组 nums = [1, 0, -1, 0, -2, 2]，和 target = 0。

满足要求的四元组集合为：
[
  [-1,  0, 0, 1],
  [-2, -1, 1, 2],
  [-2,  0, 0, 2]
]
```

### 思路

参考3sum的解法。先数组排序，使用四个指针(a<b<c<d)。固定最小的a和b在左边，c=b+1，d=n-1，移动两个指针求解。

保存nums[a]+nums[b]+nums[c]+nums[d]==target时的解。偏大时d左移，偏小时c右移。c和d相遇时，表示以当前的a和b为最小值的解已经全部求得。b++,进入下一轮循环b循环，当b循环结束后。a++，进入下一轮a循环。

```java
class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        List<List<Integer>> res = new ArrayList<>();
        if(nums == null || nums.length < 4)
            return res;
        Arrays.sort(nums);
        int n = nums.length;
        for(int a = 0; a < n - 3; a++){
            if(a > 0 && nums[a] == nums[a - 1])
                continue;
            for(int b = a + 1; b < n - 2; b++) {
                if(b > a + 1 && nums[b] == nums[b - 1])
                    continue;
                int c = b + 1, d = n - 1;
                while(c < d) {
                    int sum = nums[a] + nums[b] + nums[c] + nums[d];
                    if(sum == target) {
                        res.add(Arrays.asList(nums[a], nums[b], nums[c], nums[d]));
                        while(c < d && nums[c] == nums[c + 1])
                            c++;
                        while(c < d && nums[d] == nums[d - 1])
                            d--;
                        c++;
                        d--;
                    }
                    else if(sum < target)
                        c++;
                    else
                        d--;
                }
            }
        }
        return res;
    }
}
```


