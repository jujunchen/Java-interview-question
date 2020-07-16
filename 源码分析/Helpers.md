# Java 11中的Helpers

```java
package java.util.concurrent;

import java.util.Collection;

/** 用于 java.util.concurrent 包中*/
class Helpers {
    //私有构造函数，不可实例化
    private Helpers() {}  

    /**
     * Collection.toString() 的一种实现，适用于有锁的类。
     * 代替了以前在整个toString()过程中加锁，或者在每次调用Iterator.next()的时候加锁
     * 该方法只在调用toArray()期间加锁，以减少其他线程对访问集合时产生的影响
     * 并且遵循在加锁期间，不调用任何外部代码
     */
    static String collectionToString(Collection<?> c) {
        final Object[] a = c.toArray();
        final int size = a.length;
        if (size == 0)
            return "[]";
        int charLength = 0;

        // Replace every array element with its string representation
        for (int i = 0; i < size; i++) {
            Object e = a[i];
            // Extreme compatibility with AbstractCollection.toString()
            String s = (e == c) ? "(this Collection)" : objectToString(e);
            a[i] = s;
            charLength += s.length();
        }

        return toString(a, size, charLength);
    }

    /**
     * Like Arrays.toString(), but caller guarantees that size > 0,
     * each element with index 0 <= i < size is a non-null String,
     * and charLength is the sum of the lengths of the input Strings.
     */
    static String toString(Object[] a, int size, int charLength) {
        // assert a != null;
        // assert size > 0;

        // Copy each string into a perfectly sized char[]
        // Length of [ , , , ] == 2 * size
        final char[] chars = new char[charLength + 2 * size];
        chars[0] = '[';
        int j = 1;
        for (int i = 0; i < size; i++) {
            if (i > 0) {
                chars[j++] = ',';
                chars[j++] = ' ';
            }
            String s = (String) a[i];
            int len = s.length();
            s.getChars(0, len, chars, j);
            j += len;
        }
        chars[j] = ']';
        // assert j == chars.length - 1;
        return new String(chars);
    }

    /** Optimized form of: key + "=" + val */
    static String mapEntryToString(Object key, Object val) {
        final String k, v;
        final int klen, vlen;
        final char[] chars =
            new char[(klen = (k = objectToString(key)).length()) +
                     (vlen = (v = objectToString(val)).length()) + 1];
        k.getChars(0, klen, chars, 0);
        chars[klen] = '=';
        v.getChars(0, vlen, chars, klen + 1);
        return new String(chars);
    }

    private static String objectToString(Object x) {
        // Extreme compatibility with StringBuilder.append(null)
        String s;
        return (x == null || (s = x.toString()) == null) ? "null" : s;
    }
}
```