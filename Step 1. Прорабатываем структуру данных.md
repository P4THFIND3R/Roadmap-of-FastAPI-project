## Step 1.1 - Создание Pydantic схем
Pydantic схемы принято хранить в папке api.
```
app/
├─ src/
│  ├─ api/
│  │  ├─ schemas/
│  │  │  ├─ task.py
│  │  │  ├─ user.py
```

Листинг файла task.py, в котором описаны стандартные модели для задач.
``` python
from pydantic import BaseModel  
from datetime import datetime  
from enum import Enum  
  
  
class TaskStatus(Enum):  
    pending = "pending"  
    planned = "planned"  
    completed = "completed"  
    canceled = "canceled"  
  
  
class Task(BaseModel):  
    title: str  
    description: str | None  
    status: TaskStatus = TaskStatus.pending.value  
  
    created_at: datetime = datetime.now()  
    updated_at: datetime = datetime.now()  
    username: str | None = None  
  
  
class TaskFromDB(Task):  
    id: int | None = None  
  
  
class TaskUpdate(BaseModel):  
    status: TaskStatus  
    updated_at: datetime = datetime.now()
```

В случае, если имеется необходимость создавать объект схемы из другого объекта, свойства будут заполняться атрибутами объекта, который передается при создании.
``` python 
class Session(BaseModel):  
  model_config = ConfigDict(from_attributes=True)
```
