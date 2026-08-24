1个crate【含有Cargo.toml】就是一个编译单元。且该编译单元的类型可以为bin，也可以为lib。此外，1个crate中可以指定多个编译入口；但是其module路径组织还是以根开始。

1个项目可以含有多个crate

## 工作空间的概念
1. 可以包含一个package，以及多个workspace members
2. 也可以仅包含多个workspace members，此时为一个virtual space