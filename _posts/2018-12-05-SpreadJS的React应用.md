---
layout: post
title: SpreadJS的React应用
excerpt: "由于业务需要采购了葡萄城的SpreadJS表格控件"
categories: [leonids]
comments: true
---

##常用组件及其属性
### SpreadSheets
* rowChanged 
主要用于删除整行触发，需要判断propertyName属性
* valueChanged 
改变单元格值触发
* rangeChanged
输入公式、delete删除数据、移动单元格触发
* workbookInitialized
初始化表格控件，返回一个spread实例
{% highlight js %}
<SpreadSheets
                  backColor="aliceblue"
                  hostStyle={{ width: '600px', height: '600px' }}
                  valueChanged={(type, sheet) => this.handleRowChanged(sheet)}
                  rangeChanged={(type, sheet) => this.handleRangeChanged(sheet)}
                  workbookInitialized={spread => this.init(spread)}
                >
{% endhighlight %}
###Worksheet
* dataSource
数据源
* name
工作簿名称
### Column
{% highlight js %}
<Worksheet name="工作簿" dataSource={data} frozenColumnCount={3}>
                    {tableHead.map(item => (
                      <Column
                        dataField={item.name}
                        width={item.width || 80}
                        key={item.name}
                        formatter={item.formatter}
                        headerText={item.displayName}
                      />
                    ))}
                  </Worksheet>
{% endhighlight %}
##渲染过程
* workbookInitialized返回一个spread实例，在init方法里可以对options按需进行配置。有了spread实例后可以生成一个Excel实例对象，其中包含了常用的spread和spreadNS，以及自定义的一些常用spread方法比如搜索功能、表格自适应高度等等
* 从服务器端获取报表数据后，再根据报表保存的模板id获取预先定义好的表头模板，渲染表头时可以做一些前期工作，例如定义选项类的单元格，定义单元格的格式、公式等等
* 将报表数据赋值给Worksheet组件的dataSource属性
* 将表头模板赋值给tableHead
* 以上两步顺序一定不能出错
## 心得
* 凡是涉及到表格重绘的地方最好都用上spread.suspendPaint()和spread.resumePaint()包裹，避免频繁重绘引起卡顿
* 多重表头：表头本身是一组树形结构的数据，对表头区域进行手工的合并单元格、赋值
* spreadJS中每一列的列头显示的是中文，但实际上存取对应的是数据库中的一个字段，所以需要通过数据绑定把表格数据和字段对应起来，模板因此而生。模板在系统管理内另有维护入口，除了定义表头的真正字段和显示的中文名之外，还可以定义表格列的宽度、格式、样式等等，支持增删改



