---
title: 面向协议写 TableView
date: 2017-06-04 17:12:21
tags:
---


TableView 是常用的一个组件，又比较复杂。灵活的运用不仅能提高开发的效率，也为后期的维护带来极大的便利。使用 TableView 就避免不了要同它的数据源和协议打交道，虽然这种设计方式使得它们跟 TableView 解耦，但同时也带来数据分散不好维护的弊端。

非常典型的例子就是在 `cellForRowAt` 或者 `didSelectRowAt` 里做各种 case 判断，这种写法在后期维护很容易造成纰漏。这篇文章介绍一种面向协议的方式，优雅的解决这个问题，**不过该方法只适用于静态的 TableView**。

Tableview 所有的协议和代理方法，有一个共同点，都跟 IndexPath 相关联，我们可以尝试根据下标，将常用的一些代理/协议方法，所需要的数据封装成数据结构，之后统一管理这个结构。

将该配置结构需要提供的数据封装成协议，其中包含 cell 重用 ID，cell 类名，cell 选中的点击事件，cell 的高度等。

``` Swift
protocol CellConfigurable {
    var reuseIdentifier: String { get }
    var cellClass: AnyClass { get }
    var selection: Selectable? { get }
    var height: CGFloat { get }
    
    func configureCell(_ cell: UITableViewCell)
}
```

Selectable 是点击 Cell 事件的封装。

``` Swift
protocol Selectable {
    func didSelectedIndexPath(_ indexPath: IndexPath)
}
```

仅仅这些还不够，我们只定义了通用的数据，Cell 可以展示各式各样的内容，可以使用 ViewModel 结构，将数据和 Cell 绑定。

``` Swift
protocol CellViewModel {
    associatedtype ViewModel
    var viewModel: ViewModel? { get }
    
    func configure(viewModel: ViewModel)
}
```

让 Cell 实现 CellViewModel 协议。

根据以上两个协议，我们就可以构造配置 Cell 的数据结构了。

``` Swift
struct CellConfigurator<Cell: UITableViewCell>: CellConfigurable where Cell: CellViewModel {
    let viewModel: Cell.ViewModel
    let reuseIdentifier: String = NSStringFromClass(Cell.self)
    let cellClass: AnyClass = Cell.self
    let selection: Selectable?
    let height: CGFloat
    
    init(viewModel: Cell.ViewModel, height: CGFloat, selection: Selectable? = nil) {
        self.viewModel = viewModel
        self.height = height
        self.selection = selection
    }
    
    func configureCell(_ cell: UITableViewCell) {
        if let cell = cell as? Cell {
            cell.configure(viewModel: viewModel)
        }
    }
}
```

根据以上的铺垫，我们来完成一个基于 TableView 的界面。

![image](http://oox2n98ab.bkt.clouddn.com/protocol_tableview.png)

该界面由开关和 Detail 混合而成。首先需要做的事情是定义 ViewModel，必须包含所有样式 Cell 所需要显示的内容，同时还有控件的事件。

``` Swift
struct TableViewCellViewModel {
    let title: String
    let detail: String?
    let image: UIImage
    let isOn: Bool
    let action: ((Bool) -> Void)?
}
```
> 这里只是 Demo，具体复杂的情况，可以按照样式定义多个 Cell 和对应的 ViewModel。

这样只需要在控制器统一的创建数据源即可

``` Swift
       let viewModel = TableViewCellViewModel(title: title, detail: detail, image: image, isOn: false, action: switchAction)
       let configurator = CellConfigurator<TableViewCell>(viewModel: viewModel,
                                                               height: 44,
                                                               selection: selectionActions[index])
       dataArray.append(configurator)
```

最后使用的时候就不需要考虑具体的 Cell 类型，以及对应下标是什么了，直接：

``` Swift
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let configurator = dataArray[indexPath.row]
        let cell = tableView.dequeueReusableCell(withIdentifier: configurator.reuseIdentifier, for: indexPath)
        configurator.configureCell(cell)
        return cell
    }
```

这是完整的 Demo: [ProtocolTableView](https://github.com/iostalks/ProtocolTableView)




