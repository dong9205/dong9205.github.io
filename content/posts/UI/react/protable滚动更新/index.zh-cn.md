---
weight: 0
title: "React Antd ProTable滚动更新"
subtitle: ""
date: 2024-04-11T13:12:40+08:00
lastmod: 2024-04-11T13:12:40+08:00
draft: false
author: "Derrick"
authorLink: "https://www.p-pp.cn/"
summary: ""
license: ""
tags: [""]
resources:
- name: "featured-image"
  src: ""
categories: 
- "documentation"

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false

toc:
  enable: true
  auto: true

mapbox:
share:
  enable: true
comment:
  enable: true

lightgallery: true
---

# React Antd ProTable expandable.expandedRowRender支持滚动更新

## 需求介绍
报警平台需要展示各个报警的事件，相同的报警标题会被聚合在一起，但是会记录每次报警内容作为事件展示，有时候出现几百万次报警，导致前端加载缓慢，所以考虑报警事件使用滚动加载的方式获取

## 前端实现

### protable
```typescript
            <ProTable<API.RuleListItem, API.PageParams>
                actionRef={actionRef}
                rowKey="ID"
                search={{
                    labelWidth: "auto",
                    defaultCollapsed: true,
                }}
                scroll={{
                    x: true,
                }}
                dateFormatter={(value, valueType) => {
                    return value.format('YYYY-MM-DDTHH:mm:ss.SSS+08:00');
                }}
                form={{
                    syncToUrl: (values, type) => {
                        if (type === 'get') {
                            return {
                                ...values,
                                created_at: [values.startTime, values.endTime],
                            };
                        }
                        return values;
                    },
                }}
                headerTitle={getTitle()}
                request={async (params, sorter, filter) => {
                    let data = await GetObjects() // 获取报警列表
                    return data
                }}
                pagination={{
                    showSizeChanger: true,
                }}
                columns={columns}
                rowSelection={{
                    selections: [
                        Table.SELECTION_INVERT,
                        Table.SELECTION_NONE,
                    ],
                    preserveSelectedRowKeys: true,
                    onChange: (_, selectedRows) => {
                        setSelectedRows(selectedRows);
                    },
                }}
                expandable={{
                    expandedRowRender: record => (
                        <ExpandedRowRender record={record} />  // 将record传给expandedRowRender
                    ),
                }}
            />
```

###
```typescript
const expandedRowRender = React.memo(({ record }) => {
    const [expandedData, setExpandedData] = useState<[]>(record.events);  // 首先在获取报警列表时，会拿到报警的前10条事件，初始化事件
    const loadMoreData = () => {
        console.log("loadMoreData");
        GetEvents("events", { alarmID: record.ID, startCount: expandedData.length, pageSize: 10 }).then((res) => { // 后端offset startCount limit pageSize
            setExpandedData(expandedData => {
                return [...expandedData, ...res.data].sort((firstItem, secondItem) => secondItem.ID - firstItem.ID)  // 将已有数据和新数据进行合并并按照ID倒序展示
            })
        }
        )
    };
    return (
        <div
            id="scrollableDiv"
            style={{
                height: 300,
                overflow: 'auto',
                padding: '0 16px',
                border: '1px solid rgba(140, 140, 140, 0.35)',
            }}
        >
            <InfiniteScroll
                dataLength={expandedData.length}
                next={loadMoreData}  // 每次滚动到末尾时，会执行loadMoreData获取更多的数据.
                hasMore={expandedData.length < record.eventCount} // 是否有更多数据,eventCount会记录event的总数量，如果当前数据长度小于eventCount，说明还有更多事件
                loader={<Spin />}
                endMessage={<Divider plain>It is all, nothing more 🤐</Divider>}
                scrollableTarget="scrollableDiv"
            >
                <ProTable
                    columns={EventColumns}
                    headerTitle={false}
                    search={false}
                    options={false}
                    dataSource={expandedData}
                    pagination={false}
                    rowKey="ID"
                />
            </InfiniteScroll>
        </div >

    );
});
```

## 相关问题

### Rendered more fewer hooks than during the previous render
这个报错是因为我在expandedRowRender中使用useState的报错，因为expandedRowRender是属于retuen的内容，会有多次渲染，如果在expandedRowRender使用useState，会导致useState被渲染多次，这是React不允许的，参考: https://www.cnblogs.com/chuckQu/p/16644590.html.

后续经过和GPT的沟通，发现可以使用React.memo解决该问题

###  在使用React.memo之前，因为我的expandedRowRender是在另一个TS文件中，使用函数传递record，然后在expandedRowRender中获取新的数据，并更新主组件中的内容，但是我发现主组件内容更新后在expandedRowRender中的数据不会被重新渲染,参考的GPT代码如下

```typescript
import React, { useState, useEffect } from 'react';
import { Table } from 'antd';
import ExpandedRowComponent from './ExpandedRowComponent';

const ParentComponent = () => {
  const [expandedDataMap, setExpandedDataMap] = useState({});

  // 更新展开行的数据
  const updateExpandedRowData = (recordKey, newData) => {
    // 创建新的对象以确保引用发生变化
    const newExpandedDataMap = { ...expandedDataMap };
    newExpandedDataMap[recordKey] = newData;
    setExpandedDataMap(newExpandedDataMap);
  };
  useEffect(() => {
      const intervalId = setInterval(() => {
        console.log('Expanded Data Map:', expandedDataMap);
      }, 3000);

      return () => {
        clearInterval(intervalId);
      };
    }, [expandedDataMap]);
  const columns = [
    // 主表格的列定义
    // 例如：{ title: 'Name', dataIndex: 'name', key: 'name' }
  ];

  const data = [
    // 主表格数据
    // 例如：{ key: '1', name: 'John Brown' }
  ];

  return (
    <Table
      columns={columns}
      dataSource={data}
      expandable={{
        expandedRowRender: (record) => (
          <ExpandedRowComponent
            record={record}
            expandedData={expandedDataMap[record.key] || []} // 传递展开行数据
            updateExpandedRowData={updateExpandedRowData}
          />
        ),
        onExpand: (expanded, record) => {
          // 在展开或收起时更新状态
          if (expanded) {
            // 加载展开的数据并更新状态
            // 例如：fetchExpandedData(record.key).then(newData => updateExpandedRowData(record.key, newData));
          } else {
            // 收起时清空展开的数据状态
            updateExpandedRowData(record.key, []);
          }
        },
      }}
    />
  );
};

export default ParentComponent;
```

```typescript
// ExpandedRowComponent.js
import React from 'react';
import { Table } from 'antd';

const ExpandedRowComponent = ({ record, expandedData, updateExpandedRowData }) => {
  const handleUpdateExpandedData = () => {
    // 调用父组件传递的更新函数来更新展开行的数据
    updateExpandedRowData(record.key, /* 新数据 */);
  };

  const columns = [
    // 定义表格列
    // 例如：{ title: 'Column 1', dataIndex: 'column1', key: 'column1' }
  ];

  return (
    <Table
      columns={columns}
      dataSource={expandedData}
      pagination={false}
      // 在这里添加滚动更新的逻辑
      // 例如：onScroll={handleUpdateExpandedData}
    />
  );
};

export default ExpandedRowComponent;
```

在ExpandedRowComponent中跑了updateExpandedRowData之后，ParentComponent中每三秒打印的内容中可以看到新的数据，但是在ExpandedRowComponent中的表格并未重新渲染

## 参考文档
列表滚动加载: https://ant.design/components/list-cn#list-demo-infinite-load
protable树状数据如何实现懒加载子节点: https://github.com/ant-design/pro-components/issues/2332
滚动加载组件： https://github.com/ankeetmaini/react-infinite-scroll-component.git
