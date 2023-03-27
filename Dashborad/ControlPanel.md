> [!example]+ 命令按钮
> `button-idea`
> 


> [!example]+ 待办事项
>- [ ] RecyclerView 分析⏫ 
>- [ ] Fragment 分析
>- [ ] 协程分析
>- [ ] KMM 学习
>- [ ] Jetpack 学习


```dataview 
table content
from "Dashboard/Kanban"
where contains(file.content,"## Todo")
```


```dataviewjs  
const todosHeading = "## Todo";  
  
function extractTodos(text) {  
  const lines = text.split("\n");  
  const todosIndex = lines.findIndex(line => line.trim() === todosHeading);  
  if (todosIndex === -1) return '';  
  
  const todosSection = lines.slice(todosIndex);  
    
  for (let i = 1; i < todosSection.length; i++) {  
    if (todosSection[i].startsWith("#")) {  
      return todosSection.slice(1, i).join("\n");  
    }  
  }  
    
  return todosSection.slice(1).join("\n");  
}  
  
const files = dv.pages().where(page => page.file.content.contains(todosHeading));  
  
dv.table(["File", "Todo Section"], files.map(file => {  
  return {  
    "File": file.file.link,  
    "Todo Section": extractTodos(file.file.content)  
  };  
}));  
```