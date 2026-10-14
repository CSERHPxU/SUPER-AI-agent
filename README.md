import java.util.Arrays;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        Scanner scanner = new Scanner(System.in);

        int[] values = new int[10000];
        int n = 0;

        while (scanner.hasNextInt()) {
            values[n++] = scanner.nextInt();
        }

        if (n == 0) {
            System.out.println(-1);
            return;
        }

        int[] left = new int[n];
        int[] right = new int[n];
        Arrays.fill(left, -1);
        Arrays.fill(right, -1);

        // 栈中保存节点在 values 数组中的下标
        int[] stack = new int[n];
        int top = 0;

        int root = n - 1;
        stack[0] = root;

        long upperBound = Long.MAX_VALUE;
        boolean valid = true;

        // 从后往前：根 -> 右子树 -> 左子树
        for (int i = n - 2; i >= 0; i--) {
            int currentValue = values[i];

            // 当前节点超过所属子树允许的上界
            if (currentValue >= upperBound) {
                valid = false;
                break;
            }

            int parent = -1;

            /*
             * 当前值小于栈顶时，说明开始进入某个节点的左子树。
             * 最后一个被弹出的节点就是当前节点的父节点。
             */
            while (top >= 0 && currentValue < values[stack[top]]) {
                parent = stack[top--];
                upperBound = values[parent];
            }

            // 题目保证无重复，这里仍做防御性判断
            if (top >= 0 && currentValue == values[stack[top]]) {
                valid = false;
                break;
            }

            if (parent != -1) {
                // 当前节点是 parent 的左孩子
                left[parent] = i;
            } else {
                // 没有弹栈，当前节点是栈顶节点的右孩子
                if (top < 0) {
                    valid = false;
                    break;
                }
                right[stack[top]] = i;
            }

            stack[++top] = i;
        }

        if (!valid) {
            System.out.println(-1);
            return;
        }

        // 层序遍历，累加每层最后一个节点
        int[] queue = new int[n];
        int head = 0;
        int tail = 0;
        queue[tail++] = root;

        long answer = 0;

        while (head < tail) {
            int levelSize = tail - head;
            int rightMostValue = 0;

            for (int i = 0; i < levelSize; i++) {
                int node = queue[head++];
                rightMostValue = values[node];

                // 先左后右，因此本层最后处理的是最右节点
                if (left[node] != -1) {
                    queue[tail++] = left[node];
                }

                if (right[node] != -1) {
                    queue[tail++] = right[node];
                }
            }

            answer += rightMostValue;
        }

        System.out.println(answer);
    }
}
