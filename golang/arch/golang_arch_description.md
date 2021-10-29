# Golang Architecture Services(Clean Architecture)

![Clean Architecture](https://github.com/alfssobsd/notes/blob/main/golang/arch/%D0%90%D1%80%D1%85%D0%B8%D1%82%D0%B5%D0%BA%D1%82%D1%80%D0%B0%20GoLang%20%D0%BF%D1%80%D0%B8%D0%BB%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D0%B8%CC%86-Clean%20Architecture.jpg)

![Borders Descriptions](https://github.com/alfssobsd/notes/blob/main/golang/arch/%D0%90%D1%80%D1%85%D0%B8%D1%82%D0%B5%D0%BA%D1%82%D1%80%D0%B0%20GoLang%20%D0%BF%D1%80%D0%B8%D0%BB%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D0%B8%CC%86-Borders.jpg)

![Exaplain](https://github.com/alfssobsd/notes/blob/main/golang/arch/%D0%90%D1%80%D1%85%D0%B8%D1%82%D0%B5%D0%BA%D1%82%D1%80%D0%B0%20GoLang%20%D0%BF%D1%80%D0%B8%D0%BB%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D0%B8%CC%86-Example_%20How%20it%20works.jpg)


## Типы сервисов и их структура кода
Есть как минимум следующие типы сервисов

* Executor - основной сервис хранения бизнес логики в рамках блока системы
* Api-Gateway - сервис предоставления API для внешних систем, Web UI пользователя или Web UI BackOffice
* Scheduler - сервис постановки задач на обработку , обычно общается только с worker
* Worker - сервисы обрабатывающие background задачи

### Executor
Структура каталогов модуля
```
├── entrypoints # entrypoints
│   ├── grpc # воходная точка для grpc запросов
│   │   ├── ads_settings_server_impl.go # описательная часть должна находится в другом go модуле
│   │   └── user_server_impl.go
│   └── http # входная точка для http запросов
│       ├── ads_settings_controller.go
│       ├── ads_settings_controller_dto.go # описания DTO для request/response
│       ├── user_manage_controller.go
│       └── user_manage_controller_dto.go
├── domain # domain - в большенстве случаев не требуется, можно обойтись только usecase
│   ├── ads_comission_calc_domain.go
│   └── ads_comission_calc_domain_dto.go
├── usecases
│   ├── ads_settings_usecases.go
│   ├── ads_settings_usecases_dto.go # описания DTO для in/out
│   ├── user_manage_usecases.go
│   └── user_manage_usecases_dto.go
├── dataproviders
│   ├── authservice
│   ├── pg_provider
│   │   ├── ads_settings_repo.go # репозитории для работы с  моделями
│   │   ├── models
│   │   │   ├── gen_model.go # автоматически сгенерированные модели с помощью genna https://github.com/dizzyfool/genna
│   │   │   └── model_validation.go # валидация автоматически сгенерированных моделей
│   │   └── user_repo.go # репозитории для работы с  моделями
│   └── rabbitmq
│       └── maker_task_provider.go # провайдер для постановки задач в RMQ
├── go.mod
├── go.sum
└── main.go # основная точка входа, фактически слой configuration
```

### Api-Gateway
Структура каталогов модуля
```
├── entrypoints # entrypoints
│   └── http # входная точка для http запросов
│       ├── ads_settings_controller.go
│       └── ads_settings_controller_dto.go # описания DTO для request/response
├── usecases # занимается переупоковкой запросов и проверкой доступов
│   └── ads_settings_proxy_usecases.go
├── dataproviders
│   └── myservice
│       └── myservice_client.go # клиент для обращения к сервису
├── go.mod
├── go.sum
└── main.go # основная точка входа, фактически слой configuration
```

### Scheduler
Структура каталогов модуля
```
├── entrypoints # entrypoints
│   └──cron # запуск переодических задач
│       └── cron_contorller.go
├── usecases
│   ├── periodic_task_scheduler_usecase.go # usecase который выполняет бизнес-логику для постановки задач
│   └── periodic_task_scheduler_usecase_dto.go # описания DTO для in/out
├── dataproviders
│   ├── rabbitmq
│   │   └── maker_task_provider.go # провайдер для постановки задач в RMQ
│   └── myservice
│       └── myservice_client.go # клиент для обращения к сервису, к получение записей для формирование заданий
├── go.mod
├── go.sum
└── main.go # основная точка входа, фактически слой configuration
```

### Worker
Структура каталогов модуля
```

├── entrypoints # entrypoints
│   └──rabbitmq # воходная точка для RMQ задач
│       └── task_contorller.go
├── usecases
│   ├── somthing_task_handler_usescases.go
│   └── somthing_task_handler_usescases_dto.go # описания DTO для in/out
├── dataproviders
│   └── myservice
│       └── myservice_client.go # клиент для обращения к сервису который необходим в результате выполнения задачи
├── go.mod
├── go.sum
└── main.go # основная точка входа, фактически слой configuration
```

## Примеры кода
### UseCase
```
type adsSettingsUseCases struct {
    someRepo     pg_provider.SomeRepo
    anotherRepo  pg_provider.AnotherRepo
}
 
func NewAdsSettingsUseCases(someRepo pg_provider.SomeRepo, anotherRepo pg_provider.AnotherRepo) *adsSettingsUseCases {
 
    return &adsSettingsUseCases{someRepo: someRepo,anotherRepo: anotherRepo}
}
 
type AdsSettingsUseCases interface {
    ActionOne(ActionOneInDTO) (ActionOneOutDTO, error)
    AnotherAction(AnotherActionInDTO) (*uuid.UUID, error)
}
 
func (uc *adsSettingsUseCases) ActionOne(in ActionOneInDTO) (ActionOneOutDTO, error) {
    log.Info("AdsSettingsUseCases.ActionOne starting...")
      
    //some logic
    return ActionOneOutDTO{}, nil
}
 
func (uc *adsSettingsUseCases) AnotherAction(in AnotherActionInDTO) (*uuid.UUID, error) {
    log.Info("AdsSettingsUseCases.ActionOne starting...")
      
    //some logic
 
    return &newUUID, nil
}
```

### Entrypoints
```
func ProductRoutes(e *echo.Echo, db *sqlx.DB) {
    //create repos and usecases
    productRepository := postgres.NewProductRepository(db)
    productUseCase := usecases.NewProductUseCase(productRepository)
 
    e.GET("/api/v1/products", func(c echo.Context) error {
        return searchProductsController(c, productUseCase)
    })
    e.GET("/api/v1/products/:id", func(c echo.Context) error {
        return showProductDetailInfoController(c, productUseCase)
    })
 
    e.POST("/api/v1/products", func(c echo.Context) error {
        return createProductController(c, productUseCase)
    })
 
    e.POST("/api/v1/products/excel", func(c echo.Context) error {
        return createProductsFromExcelController(c, productUseCase)
    })
}
 
func searchProductsController(c echo.Context, productUseCase usecases.ProductUseCase) error {
    log.Info("searchProductsController")
 
    productList := productUseCase.SearchProductsUseCase()
    var responseProductList []entities.HttpProductResponseEntity
    responseProductList = []entities.HttpProductResponseEntity{}
 
    for _, element := range productList {
        responseProductList = append(responseProductList, entities.HttpProductResponseEntity{
            ProductId:          element.ProductId,
            ProductCodeName:    element.ProductCodeName,
            ProductTitle:       element.ProductTitle,
            ProductDescription: element.ProductDescrition,
            ProductPrice:       element.ProductPrice,
        })
    }
    return c.JSON(http.StatusOK, entities.HttpProductListResponseEntity{
        Total:  len(responseProductList),
        Offset: 0,
        Items:  responseProductList,
    })
}
 
func showProductDetailInfoController(c echo.Context, productUseCase usecases.ProductUseCase) error {
    id := c.Param("id")
    item, err := productUseCase.ShowProductDetailInfoUseCase(uuid.FromStringOrNil(id))
    if err != nil {
        return c.JSON(http.StatusNotFound, entities.HttpActionResponseEntity{
            Code:    http.StatusNotFound,
            Message: err.Error(),
        })
    }
    return c.JSON(http.StatusOK, entities.HttpProductResponseEntity{
        ProductId:          item.ProductId,
        ProductCodeName:    item.ProductCodeName,
        ProductTitle:       item.ProductTitle,
        ProductDescription: item.ProductDescrition,
        ProductPrice:       item.ProductPrice,
    })
}
 
func createProductController(c echo.Context, productUseCase usecases.ProductUseCase) error {
 
    r := new(entities.HttpProductRequestEntity)
    _ = c.Bind(r)
    log.Info("createProductController ", r)
 
    productEntity := productUseCase.CreateProductUseCase(_useCaseEntities.ProductUseCaseEntity{
        ProductTitle:      r.ProductTitle,
        ProductCodeName:   r.ProductCodeName,
        ProductPrice:      r.ProductPrice,
        ProductDescrition: r.ProductDescription,
    })
 
    return c.JSON(http.StatusOK, entities.HttpProductResponseEntity{
        ProductId:          productEntity.ProductId,
        ProductCodeName:    productEntity.ProductCodeName,
        ProductTitle:       productEntity.ProductTitle,
        ProductDescription: productEntity.ProductDescrition,
        ProductPrice:       productEntity.ProductPrice,
    })
}
 
func createProductsFromExcelController(c echo.Context, productUseCase usecases.ProductUseCase) error {
    log.Info("createProductsFromExcelController ")
 
    //prepare temp file for parsing
    file, err := c.FormFile("file")
    if err != nil {
        return c.JSON(http.StatusBadRequest, err)
    }
 
    src, err := file.Open()
    if err != nil {
        return c.JSON(http.StatusBadRequest, err)
    }
 
    defer func() {
        _ = src.Close()
    }()
 
    tmpfile, err := ioutil.TempFile("", "products.*.xlsx")
    if err != nil {
        return c.JSON(http.StatusBadRequest, err)
    }
 
    if _, err = io.Copy(tmpfile, src); err != nil {
        return c.JSON(http.StatusBadRequest, err)
    }
 
    productsList := productUseCase.CreateProductFromExcelUseCase(tmpfile.Name())
    var responseList []entities.HttpProductResponseEntity
    for _, element := range productsList {
        responseList = append(responseList, entities.HttpProductResponseEntity{
            ProductId:          element.ProductId,
            ProductCodeName:    element.ProductCodeName,
            ProductTitle:       element.ProductTitle,
            ProductDescription: element.ProductDescrition,
            ProductPrice:       element.ProductPrice,
        })
    }
    return c.JSON(http.StatusOK, entities.HttpProductListResponseEntity{
        Total:  len(responseList),
        Offset: 0,
        Items:  responseList,
    })
}
```
