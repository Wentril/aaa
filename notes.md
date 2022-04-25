# prepare second controler
* * client/controller.py - setup
```Python
"""
Client
"""
from fastapi import APIRouter, Request, Form
from fastapi.responses import HTMLResponse

from fastapi.templating import Jinja2Templates

import requests

client_router = APIRouter(prefix="", tags=["web_client"])

templates = Jinja2Templates(directory="warehouse_app/client/templates")
```
* warehouse_app/app.py - routers
```Python
"""
Warehouse application
"""
from fastapi import FastAPI

from warehouse_app.warehouse.controller import warehouse_router
from warehouse_app.client.controller import client_router

app = FastAPI(title="Warehouse API")

app.include_router(warehouse_router)
app.include_router(client_router)


@app.get("/")
def root():
    return {"message": "Hello, this route is useless for now!"}
```
# basic example
* example.html
```html
<!DOCTYPE html>
<html>
<body>

<h1>My First Heading</h1>
<p>My first paragraph.</p>

</body>
</html>
```
* client/controller.py - add endpoint
```Python
@client_router.get("/example", response_class=HTMLResponse)
def list_warehouses(request: Request):
    """ Example """

    return templates.TemplateResponse("example.html", {"request": request})
```

# warehouse listing
* database/repository.py - add get_all_warehouses to database
```Python
def get_all_warehouses(self) -> list[WarehouseBto]:
    """ Gets all unique names of warehouses
    Returns:
        list of names (strings)
    """
    with self._Session() as session:
        stmt = select(Warehouse)

        res = session.scalars(stmt)

        return [WarehouseBto.bto(w) for w in res]
```
* warehouse/dto.py - upgrade dto models
```Python
class WarehouseItemDto(BaseModel):
    name: str
    amount: int

    def model(self) -> ItemBto:
        item = ItemBto(
            name=self.name,
            amount=self.amount,
        )
        return item

    @staticmethod
    def dto(item: ItemBto) -> 'WarehouseItemDto':
        return WarehouseItemDto(
            name=item.name,
            amount=item.amount,
        )
```
```Python
class WarehouseDto(BaseModel):
    name: str
    location: str
    capacity: int
    items: List[WarehouseItemDto]

    def model(self) -> WarehouseBto:
        warehouse = WarehouseBto(
            name=self.name,
            location=self.location,
            capacity=self.capacity,
            items=[item.model() for item in self.items],
        )
        return warehouse

    @staticmethod
    def dto(warehouse: WarehouseBto) -> 'WarehouseDto':
        return WarehouseDto(
            name=warehouse.name,
            location=warehouse.location,
            capacity=warehouse.capacity,
            items=[WarehouseItemDto.dto(item) for item in warehouse.items]
        )
```
* warehouse/controller.py - add endpoint
```Python
@warehouse_router.get("/warehouse", response_model=List[WarehouseDto])
def get_warehouses():
    """ Gets all warehouses
    Returns:
        list of warehouses (WarehouseDto), STATUS 200 (list can be empty)
    """
    warehouses = [WarehouseDto.dto(w) for w in warehouse_database.get_all_warehouses()]
    return warehouses
```
* client/controller.py - add endpoint
```Python
@client_router.get("/warehouse_list", response_class=HTMLResponse)
def list_warehouses(request: Request):
    """ List of all warehouses """

    path = 'warehouse/warehouse/'

    response = requests.get(url=HOST+path)

    return templates.TemplateResponse("warehouse_list.html", {"request": request, "warehouses": response.json()})
```
* warehouse_list.html - table, jinja, header 
```html
<!DOCTYPE html>
<html>
<body>
    <h3 class="card-header text-center">Warehouse list</h3>

    <table class="table table-striped table-hover caption-top">
        <thead>
            <tr>
                <th scope="col">#</th>
                <th>Name</th>
                <th>Location</th>
                <th>Capacity</th>
                <th>Items</th>
            </tr>
        </thead>
        <tbody>
            {% for warehouse in warehouses %}
                <tr>
                    <th scope="col">{{ loop.index }}</th>
                    <td>{{ warehouse['name'] }}</td>
                    <td>{{ warehouse['location'] }}</td>
                    <td>{{ warehouse['capacity'] }}</td>
                    <td>
                        {% for item in warehouse['items'] %}
                            {{ item['name'] }} ({{ item['amount'] }}),
                        {% endfor %}
                    </td>
                </tr>
            {% endfor %}
        </tbody>
    </table>
    </body>
</html>
```
# warehouse adding
* client/controller.py - add get endpoint
```Python
@client_router.get("/warehouse_add", response_class=HTMLResponse)
def add_warehouses(request: Request):
    """ Add a new warehouse to database """

    return templates.TemplateResponse("warehouse_add.html", {"request": request})
```
* warehouse_list.html - table, jinja, header 
```html
<!DOCTYPE html>
<html>
<body>
    <div class="col-sm-auto">
        <div class="card">
            <h3 class="card-header text-center">
                Add new warehouse
            </h3>
            <div class="card-body mt-3">
                <form method="post">
                    <div class="row">
                        <label for="name" class="form-label">Name</label >
                        <input type="text" name="name" class="form-control form-control-lg" id="name" required>
                    </div>
                    <div class="row">
                        <label for="location" class="form-label">Location</label >
                        <input type="text" name="location" class="form-control form-control-lg" id="location" required>
                    </div>
                    <div class="row">
                        <label for="capacity" class="form-label">Capacity</label >
                        <input type="number" name="capacity" class="form-control form-control-lg" id="capacity" required>
                    </div>
                    <div class="card-body text-center">
                        <button class="btn btn-lg btn-primary btn-block mt-3 text-center" type="submit">Confirm</button>
                    </div>
                </form>
            </div>
        </div>
    </div>
</body>
</html>
```
* client/controller.py - add get endpoint
```Python
@client_router.post("/warehouse_add", response_class=HTMLResponse, status_code=200)
def add_warehouses(request: Request, name: str = Form(...), location: str = Form(...), capacity: str = Form(...)):
    """ Add a new warehouse to database """

    path = 'warehouse/'

    data = {
        'name': name,
        'location': location,
        'capacity': capacity,
        'items': []
    }

    response = requests.post(url=HOST+path, json=data)

    if not response.ok:
        return templates.TemplateResponse("error.html", {"request": request, "status_code": response.status_code, "error_message": response.json()['detail']})

    return templates.TemplateResponse("warehouse_add.html", {"request": request})
```
# navigation
* add navigation between pages
```html
<body>
    <div class="card">
        <nav class="nav">
            <a class="nav-link" href="http://127.0.0.1:8000/warehouse_list">Warehouse list</a>
            <a class="nav-link" href="http://127.0.0.1:8000/warehouse_add">Warehouse add</a>
            <a class="nav-link" href="http://127.0.0.1:8000/item_list">Item list</a>
            <a class="nav-link" href="http://127.0.0.1:8000/item_add">Item add</a>
        </nav>
    </div>
...
```

# style
* add style
```html
    <head>
        <!-- Required meta tags -->
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">

        <!-- Bootstrap CSS -->
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC" crossorigin="anonymous">

        <title>{% block title %}{% endblock %}</title>
    </head>
```

# templating
* base.html
```html
<!doctype html>
<html lang="en">
    <head>
        <!-- Required meta tags -->
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">

        <!-- Bootstrap CSS -->
        <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet" integrity="sha384-EVSTQN3/azprG1Anm3QDgpJLIm9Nao0Yz1ztcQTwFspd3yD65VohhpuuCOmLASjC" crossorigin="anonymous">

        <title>{% block title %}{% endblock %}</title>
    </head>
    <body>
        <div class="card">
            <nav class="nav">
                <a class="nav-link" href="http://127.0.0.1:8000/warehouse_list">Warehouse list</a>
                <a class="nav-link" href="http://127.0.0.1:8000/warehouse_add">Warehouse add</a>
                <a class="nav-link" href="http://127.0.0.1:8000/item_list">Item list</a>
                <a class="nav-link" href="http://127.0.0.1:8000/item_add">Item add</a>
            </nav>
        </div>
        <div class="container-fluid">
            <div class="card-body mt-4">
                <div class="row justify-content-center">
                    {% block content %}
                    {% endblock %}
                </div>
            </div>
        </div>
    </body>
</html>
```
* update others
```html
{% extends "base.html" %}
{% block title %}Warehouse list{% endblock %}

{% block content %}
    <h3 class="card-header text-center">Warehouse list</h3>

    <table class="table table-striped table-hover caption-top">
        <thead>
            <tr>
                <th scope="col">#</th>
                <th>Name</th>
                <th>Location</th>
                <th>Capacity</th>
                <th>Items</th>
            </tr>
        </thead>
        <tbody>
            {% for warehouse in warehouses %}
                <tr>
                    <th scope="col">{{ loop.index }}</th>
                    <td>{{ warehouse['name'] }}</td>
                    <td>{{ warehouse['location'] }}</td>
                    <td>{{ warehouse['capacity'] }}</td>
                    <td>
                        {% for item in warehouse['items'] %}
                            {{ item['name'] }} ({{ item['amount'] }}),
                        {% endfor %}
                    </td>
                </tr>
            {% endfor %}
        </tbody>
    </table>
{% endblock %}
```
```html
{% extends "base.html" %}
{% block title %}Add warehouse{% endblock %}

{% block content %}
    <div class="col-sm-auto">
        <div class="card">
            <h3 class="card-header text-center">
                Add new warehouse
            </h3>
            <div class="card-body mt-3">
                <form method="post">
                    <div class="row">
                        <label for="name" class="form-label">Name</label >
                        <input type="text" name="name" class="form-control form-control-lg" id="name" required>
                    </div>
                    <div class="row">
                        <label for="location" class="form-label">Location</label >
                        <input type="text" name="location" class="form-control form-control-lg" id="location" required>
                    </div>
                    <div class="row">
                        <label for="capacity" class="form-label">Capacity</label >
                        <input type="number" name="capacity" class="form-control form-control-lg" id="capacity" required>
                    </div>
                    <div class="card-body text-center">
                        <button class="btn btn-lg btn-primary btn-block mt-3 text-center" type="submit">Confirm</button>
                    </div>
                </form>
            </div>
        </div>
    </div>
{% endblock %}
```