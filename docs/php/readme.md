### PHP扩展开发
  PHP为了扩展方便扩展开发，提供了一个ext_skel工具，可以自动生成扩展代码骨架。
  
  首先我们假设要创建一个venus扩展，其中提供一个函数venus_skel函数， 这个原型文件为venus.skel, 代码如下:
```
string venus_string(string str)
```
  那么我们可以使用ext_skel工具来生成这个扩展的骨架。`./ext_skel --extname=venus --proto=venus.skel`
  
  
