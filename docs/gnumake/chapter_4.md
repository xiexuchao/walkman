## 函数

  GNU make支持内置函数以及用户自定义函数。函数调用看起来非常像变量引用，不过前者包含了一个或多个被都好隔开的参数。大部分内置函数扩展后多半会被赋值给一个变量或是传递给一个subshell. 用户自定义函数则存储在变量或宏中，而且会接收调用者传来的一个或多个参数。
  
### 用户自定义函数
  用户自定义函数能够将命令序列存储在变量里，让我们得以在makefile中使用各种应用程序。在下面的例子中我们可以看到一个用来终止进程的宏:
```
AWK   := awk
KILL  := kill

# $(kill-acroread)
define kill-acroread
  @ ps -W |                                         \
  $(AWK) 'BEGIN {FIELDWIDTH = "9 47 100* }          \
          /AcroRd32/ {                              \
                        print "Killing " $$3;       \
                        system( "$(KILL) -f " $$1)  \
                      }'
endef
```
