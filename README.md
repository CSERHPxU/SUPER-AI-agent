import java.io.*;
import java.math.BigInteger;

public class Main {
    private static final class FastScanner {
        private final InputStream in = System.in;
        private final byte[] buffer = new byte[1 << 16];
        private int ptr = 0, len = 0;

        private int read() throws IOException {
            if (ptr >= len) {
                len = in.read(buffer);
                ptr = 0;
                if (len <= 0) return -1;
            }
            return buffer[ptr++];
        }

        long nextLong() throws IOException {
            int c;
            do {
                c = read();
            } while (c <= ' ' && c != -1);

            long sign = 1;
            if (c == '-') {
                sign = -1;
                c = read();
            }

            long value = 0;
            while (c > ' ') {
                value = value * 10 + (c - '0');
                c = read();
            }
            return value * sign;
        }
    }

    private static final class Node {
        long price;
        long quantity;
        final int id;
        final long priority;

        Node left, right;
        long lazyAdd;

        // The smallest/largest composite key in this subtree.
        long minKeyPrice, maxKeyPrice;
        int minKeyId, maxKeyId;

        Node(long price, long quantity, int id) {
            this.price = price;
            this.quantity = quantity;
            this.id = id;
            this.priority = nextPriority();
            this.minKeyPrice = this.maxKeyPrice = price;
            this.minKeyId = this.maxKeyId = id;
        }
    }

    private static final class Pair {
        Node left, right;

        Pair(Node left, Node right) {
            this.left = left;
            this.right = right;
        }
    }

    private static long randomSeed = 0x1234ABCD5678EF90L;

    private static long nextPriority() {
        // SplitMix64: deterministic and sufficiently well distributed for a treap.
        randomSeed += 0x9E3779B97F4A7C15L;
        long z = randomSeed;
        z = (z ^ (z >>> 30)) * 0xBF58476D1CE4E5B9L;
        z = (z ^ (z >>> 27)) * 0x94D049BB133111EBL;
        return z ^ (z >>> 31);
    }

    /*
     * Composite-key order:
     *   1. lower price comes first;
     *   2. at the same price, larger id comes first.
     * Therefore the rightmost eligible node is the earliest order (smallest id)
     * among all orders having the best price.
     */
    private static int compareKey(long p1, int id1, long p2, int id2) {
        int byPrice = Long.compare(p1, p2);
        if (byPrice != 0) return byPrice;
        return Integer.compare(id2, id1);
    }

    private static void applyAdd(Node node, long delta) {
        if (node == null || delta == 0) return;
        node.price += delta;
        node.minKeyPrice += delta;
        node.maxKeyPrice += delta;
        node.lazyAdd += delta;
    }

    private static void push(Node node) {
        if (node == null || node.lazyAdd == 0) return;
        applyAdd(node.left, node.lazyAdd);
        applyAdd(node.right, node.lazyAdd);
        node.lazyAdd = 0;
    }

    private static void pull(Node node) {
        if (node.left != null) {
            node.minKeyPrice = node.left.minKeyPrice;
            node.minKeyId = node.left.minKeyId;
        } else {
            node.minKeyPrice = node.price;
            node.minKeyId = node.id;
        }

        if (node.right != null) {
            node.maxKeyPrice = node.right.maxKeyPrice;
            node.maxKeyId = node.right.maxKeyId;
        } else {
            node.maxKeyPrice = node.price;
            node.maxKeyId = node.id;
        }
    }

    // Requires every key in a to be smaller than every key in b.
    private static Node merge(Node a, Node b) {
        if (a == null) return b;
        if (b == null) return a;

        if (a.priority > b.priority) {
            push(a);
            a.right = merge(a.right, b);
            pull(a);
            return a;
        } else {
            push(b);
            b.left = merge(a, b.left);
            pull(b);
            return b;
        }
    }

    // Split by the complete composite key: (< key, >= key).
    private static Pair splitByKey(Node root, long price, int id) {
        if (root == null) return new Pair(null, null);
        push(root);

        if (compareKey(root.price, root.id, price, id) < 0) {
            Pair rest = splitByKey(root.right, price, id);
            root.right = rest.left;
            pull(root);
            rest.left = root;
            return rest;
        } else {
            Pair rest = splitByKey(root.left, price, id);
            root.left = rest.right;
            pull(root);
            rest.right = root;
            return rest;
        }
    }

    // Split only by price: (price < limit, price >= limit).
    private static Pair splitByPrice(Node root, long limit) {
        if (root == null) return new Pair(null, null);
        push(root);

        if (root.price < limit) {
            Pair rest = splitByPrice(root.right, limit);
            root.right = rest.left;
            pull(root);
            rest.left = root;
            return rest;
        } else {
            Pair rest = splitByPrice(root.left, limit);
            root.left = rest.right;
            pull(root);
            rest.right = root;
            return rest;
        }
    }

    // Union of two treaps whose key ranges may overlap.
    private static Node unite(Node a, Node b) {
        if (a == null) return b;
        if (b == null) return a;

        // Very common fast paths: after a bulk price increase, the two ranges
        // are often still disjoint (or have completely swapped positions).
        if (compareKey(a.maxKeyPrice, a.maxKeyId, b.minKeyPrice, b.minKeyId) < 0) {
            return merge(a, b);
        }
        if (compareKey(b.maxKeyPrice, b.maxKeyId, a.minKeyPrice, a.minKeyId) < 0) {
            return merge(b, a);
        }

        if (a.priority > b.priority) {
            push(a);
            Pair parts = splitByKey(b, a.price, a.id);
            a.left = unite(a.left, parts.left);
            a.right = unite(a.right, parts.right);
            pull(a);
            return a;
        } else {
            push(b);
            Pair parts = splitByKey(a, b.price, b.id);
            b.left = unite(parts.left, b.left);
            b.right = unite(parts.right, b.right);
            pull(b);
            return b;
        }
    }

    private static Node insert(Node root, Node node) {
        if (root == null) return node;
        push(root);

        if (node.priority > root.priority) {
            Pair parts = splitByKey(root, node.price, node.id);
            node.left = parts.left;
            node.right = parts.right;
            pull(node);
            return node;
        }

        if (compareKey(node.price, node.id, root.price, root.id) < 0) {
            root.left = insert(root.left, node);
        } else {
            root.right = insert(root.right, node);
        }
        pull(root);
        return root;
    }

    private static Node erase(Node root, long price, int id) {
        if (root == null) return null;
        push(root);
        int cmp = compareKey(price, id, root.price, root.id);

        if (cmp == 0) return merge(root.left, root.right);
        if (cmp < 0) root.left = erase(root.left, price, id);
        else root.right = erase(root.right, price, id);
        pull(root);
        return root;
    }

    // Highest price <= buyerPrice; for a price tie, earliest id is returned.
    private static Node bestSeller(Node root, long buyerPrice) {
        Node answer = null;
        Node current = root;
        while (current != null) {
            push(current);
            if (current.price <= buyerPrice) {
                answer = current;
                current = current.right;
            } else {
                current = current.left;
            }
        }
        return answer;
    }

    public static void main(String[] args) throws Exception {
        FastScanner fs = new FastScanner();
        long increase = fs.nextLong();
        int sellerCount = (int) fs.nextLong();

        Node root = null;
        for (int id = 0; id < sellerCount; id++) {
            long price = fs.nextLong();
            long quantity = fs.nextLong();
            if (quantity > 0) root = insert(root, new Node(price, quantity, id));
        }

        int buyerCount = (int) fs.nextLong();
        BigInteger answer = BigInteger.ZERO;

        for (int i = 0; i < buyerCount; i++) {
            long buyerPrice = fs.nextLong();
            long buyerQuantity = fs.nextLong();

            while (buyerQuantity > 0) {
                Node seller = bestSeller(root, buyerPrice);
                if (seller == null) break;

                long dealPrice = seller.price;
                int sellerId = seller.id;
                long dealQuantity = Math.min(buyerQuantity, seller.quantity);

                answer = answer.add(
                        BigInteger.valueOf(dealPrice).multiply(BigInteger.valueOf(dealQuantity))
                );
                buyerQuantity -= dealQuantity;
                seller.quantity -= dealQuantity;

                if (seller.quantity == 0) {
                    root = erase(root, dealPrice, sellerId);
                }

                // Every successful matching decision triggers the seller's right.
                // Sellers at exactly dealPrice are deliberately excluded.
                if (increase != 0 && root != null) {
                    Pair parts = splitByPrice(root, dealPrice);
                    applyAdd(parts.left, increase);
                    root = unite(parts.left, parts.right);
                }
            }
        }

        System.out.println(answer);
    }
}
