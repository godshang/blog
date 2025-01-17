---
title: 'Hadoop中数据文件的Join'
layout: post
categories: 技术
tags:
    - Hadoop
---


数据分析中难免要从多个数据源中提取数据并进行关联，这在数据库中通过表之间的联接即可实现，而大多数的数据库会自动的提
供对联接的处理。不过在Hadoop中数据的联接比较复杂，可以有Reduce侧联接、Map侧联接等多种方式。这里介绍一种直接通过MapReduce程序，手动的控制数据的联接。

数据来源

数据来源于《Hadoop实战》一书中的5.2节，使用一对极小的数据集。一个以逗号分隔的Customers文件，包含3个域：CustomerID、Name和Phone Number。文件内容为：

```
1,Stephanie Leung,555-555-5555
2,Edward Kim,123-456-7890
3,Jose Madriz,281-330-8004
4,David Stork,408-555-0000
```

另一个文件Orders中存储顾客的订单，有4个域：CustomerID、OrderID、Price和Purchase Date。文件内容为：

```
3,A,12.95,02-Jun-2008
1,B,88.25,20-May-2008
2,C,32.00,30-Nov-2007
3,D,25.02,22-Jan-2009
```

处理Join的思路

将Join key当作map的输出key, 也就是reduce的输入key, 这样只要join的key相同，shuffle过后，就会进入到同一个reduce 的key – value list 中去。需要为join的2张表设计一个通用的一个bean.  并且bean中加一个flag的标志属性，这样可以根据flag来区分是哪张表的数据。reduce 阶段根据flag来判断是Customers表还是Orders表就很容易了 。而join的真正处理是在reduce阶段。

代码

存储数据的Bean：CustOrder

```
public class CustOrder implements WritableComparable {

    private String customerId;
    private String customerName;
    private String orderId;
    private String orderPrice;
    private String purchaseDate;

    private int flag = 0;

    public CustOrder() {
        super();
    }

    public CustOrder(CustOrder obj) {
        this.customerId = obj.customerId;
        this.customerName = obj.customerName;
        this.orderId = obj.orderId;
        this.orderPrice = obj.orderPrice;
        this.purchaseDate = obj.purchaseDate;
        this.flag = obj.flag;
    }

    @Override
    public void readFields(DataInput in) throws IOException {
        this.customerId = in.readUTF();
        this.customerName = in.readUTF();
        this.orderId = in.readUTF();
        this.orderPrice = in.readUTF();
        this.purchaseDate = in.readUTF();
        this.flag = in.readInt();
    }

    @Override
    public void write(DataOutput out) throws IOException {
        out.writeUTF(this.customerId);
        out.writeUTF(this.customerName);
        out.writeUTF(this.orderId);
        out.writeUTF(this.orderPrice);
        out.writeUTF(this.purchaseDate);
        out.writeInt(this.flag);
    }

    @Override
    public int compareTo(Object o) {
        return 0;
    }

    public String getCustomerId() {
        return customerId;
    }

    public void setCustomerId(String customerId) {
        this.customerId = customerId;
    }

    public String getCustomerName() {
        return customerName;
    }

    public void setCustomerName(String customerName) {
        this.customerName = customerName;
    }

    public String getOrderId() {
        return orderId;
    }

    public void setOrderId(String orderId) {
        this.orderId = orderId;
    }

    public String getOrderPrice() {
        return orderPrice;
    }

    public void setOrderPrice(String orderPrice) {
        this.orderPrice = orderPrice;
    }

    public String getPurchaseDate() {
        return purchaseDate;
    }

    public void setPurchaseDate(String purchaseDate) {
        this.purchaseDate = purchaseDate;
    }

    public int getFlag() {
        return flag;
    }

    public void setFlag(int flag) {
        this.flag = flag;
    }

    public String toString() {
        return this.customerId + "," + this.customerName + "," + this.orderId + "," + this.orderPrice + ","
                + this.purchaseDate;

    }
}
```

Mapper

```
public class MapClass extends MapReduceBase implements Mapper<LongWritable, Text, Text, CustOrder> {

    private Text keyText = new Text();
    CustOrder obj = new CustOrder();

    @Override
    public void map(LongWritable key, Text value, OutputCollector<Text, CustOrder> output, Reporter reporter)
            throws IOException {
        String line = value.toString();
        String[] array = line.split(",");
        if (array.length == 3) {
            // Customer
            obj.setCustomerId(array[0]);
            obj.setCustomerName(array[1]);
            obj.setOrderId("");
            obj.setOrderPrice("");
            obj.setPurchaseDate("");
            obj.setFlag(1);
            keyText.set(obj.getCustomerId());
            output.collect(keyText, obj);
        } else if (array.length == 4) {
            // Order
            obj.setCustomerId(array[0]);
            obj.setCustomerName("");
            obj.setOrderId(array[1]);
            obj.setOrderPrice(array[2]);
            obj.setPurchaseDate(array[3]);
            obj.setFlag(2);
            keyText.set(obj.getCustomerId());
            output.collect(keyText, obj);
        }
    }

}
```

Reduce

```
public class Reduce extends MapReduceBase implements Reducer<Text, CustOrder, Text, CustOrder> {

    private CustOrder custOrder = null;
    private List<CustOrder> list = new ArrayList<CustOrder>();

    @Override
    public void reduce(Text key, Iterator<CustOrder> values, OutputCollector<Text, CustOrder> output, Reporter reporter)
            throws IOException {
        while (values.hasNext()) {
            CustOrder obj = values.next();
            if (obj.getFlag() == 1) {
                // Customer
                custOrder = new CustOrder(obj);
            } else if (obj.getFlag() == 2) {
                // Order
                CustOrder tmp = new CustOrder(obj);
                list.add(tmp);
            }
        }

        for (int i = 0; i < list.size(); i++) {
            CustOrder c = list.get(i);
            c.setCustomerName(custOrder.getCustomerName());
            output.collect(key, c);
        }
        list.clear();
    }

}
```

驱动

```
public class DemoDriver {

    public static void main(String[] args) throws IOException {
        JobConf conf = new JobConf(DemoDriver.class);
        conf.setJobName("JoinDemo");
        FileInputFormat.addInputPath(conf, new Path(args[0]));
        FileOutputFormat.setOutputPath(conf, new Path(args[1]));
        conf.setMapperClass(MapClass.class);
        conf.setReducerClass(Reduce.class);
        conf.setMapOutputKeyClass(Text.class);
        conf.setMapOutputValueClass(CustOrder.class);
        conf.setOutputKeyClass(Text.class);
        conf.setOutputValueClass(CustOrder.class);
        JobClient.runJob(conf);
    }

}
```

执行结果

```
1    1,Stephanie Leung,B,88.25,20-May-2008
2    2,Edward Kim,C,32.00,30-Nov-2007
3    3,Jose Madriz,A,12.95,02-Jun-2008
3    3,Jose Madriz,D,25.02,22-Jan-2009
```