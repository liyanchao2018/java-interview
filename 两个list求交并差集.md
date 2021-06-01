

#### 两个list如何求交差并集



~~~java

public class TestCollection {

    /**
     * 交集1: [a, b, c]
     * 交集2：[a, b, c]
     * 并集: [a, b, c, d, e, f, g, h, a, b, c, i, j, a]
     * 无重并集: [a, b, c, i, j, a, d, e, f, g, h]
     * 差集1: [d, e, f, g, h]
     * 差集2：[d, e, f, g, h]
     */


    public static void main(String[] args) {
        List<String> stringList = new ArrayList<>();
        stringList.add("a");
        stringList.add("b");
        stringList.add("c");
        stringList.add("i");
        stringList.add("j");
        stringList.add("a");//abcija

        //一、求交集
        //方法1：直接通过retainAll直接过滤
        List<String> stringList1 =  new ArrayList<>(Arrays.asList("a,b,c,d,e,f,g,h".split(",")));
        stringList1.retainAll(stringList);
        System.out.println("交集1: " + stringList1);

        //方法2：通过过滤掉存在于stringList的数据
        List<String> stringList1_2 = new ArrayList<>(Arrays.asList("a,b,c,d,e,f,g,h".split(",")));
        List<String> strings = stringList1_2.stream()
                .filter(item -> stringList.contains(item))
                .collect(toList());
        System.out.println("交集2：" + strings);

        //二、并集
        //有重并集
        List<String> stringList2 =  new ArrayList<>(Arrays.asList("a,b,c,d,e,f,g,h".split(",")));
        stringList2.addAll(stringList);
        System.out.println("并集: " + stringList2);

        //无重并集
        List<String> stringList2_2 =  new ArrayList<>(Arrays.asList("a,b,c,d,e,f,g,h".split(",")));
        List<String> stringList_1 =  new ArrayList<>(Arrays.asList("a,b,c,i,j,a".split(",")));
        stringList2_2.removeAll(stringList_1);
        stringList_1.addAll(stringList2_2);

        System.out.println("无重并集: " + stringList_1);

        //三、求差集
        //方法1：直接使用removeAll()方法
        List<String> stringList3 =  new ArrayList<>(Arrays.asList("a,b,c,d,e,f,g,h".split(",")));
        stringList3.removeAll(stringList);
        System.out.println("差集1: " + stringList3);

        /*stringList.removeAll(stringList3);
        System.out.println("stringList差集1: " + stringList);//[i, j]*/



        //方法2：通过过滤掉不存在于stringList的数据，然后和本数组进行交集处理
        List<String> stringList3_2 = new ArrayList<>(Arrays.asList("a,b,c,d,e,f,g,h".split(",")));
        System.out.println(stringList3_2.stream().filter(item -> !stringList.contains(item)).collect(toList()));
       /* stringList3_2.retainAll(stringList3_2.stream().filter(item -> !stringList.contains(item)).collect(toList()));
        System.out.println("差集2：" + stringList3_2);*/

    }

}
~~~

