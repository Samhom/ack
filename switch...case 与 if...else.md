## switch...case 与 if...else

1. 当分支较多时，当时用 switch 的效率是很高的。因为 switch 是随机访问的，就是确定了选择值之后直接跳转到那个特定的分支，但是 if...else 是遍历所以得可能值，知道找到符合条件的分支。如此看来，switch 的效率确实比 if...else 要高的多。
2. 由汇编代码可知道，switch...case 占用较多的代码空间，因为它要生成跳表，特别是当 case 常量分布范围很大但实际有效值又比较少的情况，switch...case 的空间利用率将变得很低。

3. switch...case 只能处理 case 为常量的情况，对非常量的情况是无能为力的。例如 if (a > 1 && a < 100)，是无法使用 switch...case 来处理的。所以，switch 只能是在常量选择分支时比 if...else 效率高，但是 if...else 能应用于更多的场合，ifelse比较灵活。