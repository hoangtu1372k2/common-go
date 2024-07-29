# Introduction 
Provide common golang function for other golang projects such as: auth, telemetry, reposity ... 

# Getting Started

1. To install the library, use go get:
```
go get github.com/hoangtu1372k2/common-go
```

2. Import comongo into other project
```
import (
	"git.basesystem.one/basesys/comongo/auth"
	"git.basesystem.one/basesys/comongo/reposity"
	"git.basesystem.one/basesys/comongo/telemetry"
)
```
# Connect and create tables
1. Connect to postgresql database
```
	err := reposity.Connect(
		sql_host,
		sql_port,
		sql_dbname",
		sql_sslmode",
		sql_user",
		sql_password",
		sql_schema",
	)
	if err != nil {
		return fmt.Errorf("could not establish database connection - %s", err)
	}
```

2. Create a new struct
```
	package models

	type Event struct {
		ID          uuid.UUID `json:"id" gorm:"primary_key;type:uuid;default:uuid_generate_v4()"`
		Name        string    `json:"name"`
	}

	type DTO_Event struct {
		ID          uuid.UUID `json:"id"`
		Name        string    `json:"name"`
	}
```

3. Create a table in the database
```
	err := reposity.Migrate(
				&models.Event{},
	)
	if err != nil {
		panic("Failed to AutoMigrate table! err: " + err.Error())
	}
```

# Example
1. Response controller
```
package models

// Response for one DTO
type JsonDTORsp[M any] struct {
	Code    int64  `json:"code"`
	Data    M      `json:"data,omitempty"`
	Message string `json:"message,omitempty"`
}

// Response for list of DTO with paging
type JsonDTOListRsp[M any] struct {
	Code    int64  `json:"code"`
	Count   int64  `json:"count"`
	Data    []M    `json:"data"`
	Message string `json:"message,omitempty"`
	Page    int64  `json:"page"`
	Size    int64  `json:"size"`
}

type JsonDTOListRspEvent[M any] struct {
	Code    int64         `json:"code"`
	Data    PagingData[M] `json:"data"`
	Message string        `json:"message,omitempty"`
}

func NewJsonDTORsp[M any]() *JsonDTORsp[M] {
	var dto M
	return &JsonDTORsp[M]{
		Code:    0,
		Data:    dto,
		Message: "Success",
	}
}

func NewJsonDTOListRsp[M any]() *JsonDTOListRsp[M] {
	dtoList := make([]M, 0)
	return &JsonDTOListRsp[M]{
		Code:    0,
		Count:   0,
		Data:    dtoList,
		Message: "Success",
		Size:    0,
		Page:    1,
	}
}

type PagingData[M any] struct {
	Count     int64 `json:"count"`
	Rows      []M   `json:"rows"`
	Page      int64 `json:"page"`
	Limit     int64 `json:"limit"`
	TotalPage int64 `json:"totalPage"`
}
```

2. Controller
```
package controllers

// CreateEvent		godoc
// @Summary      	Create a new event
// @Description  	Takes a event JSON and store in DB. Return saved JSON.
// @Tags         	events
// @Produce			json
// @Param        	Event  body   models.DTO_Event true  "Event JSON"
// @Success      	200   {object}  models.JsonDTORsp[models.DTO_Event]
// @Router       	/events [post]
// @Security		BearerAuth
func CreateEvent(c *gin.Context) {

	jsonRsp := models.NewJsonDTORsp[models.DTO_Event]()

	// Call BindJSON to bind the received JSON to
	var dto models.DTO_Event
	if err := c.BindJSON(&dto); err != nil {
		jsonRsp.Code = http.StatusBadRequest
		jsonRsp.Message = err.Error()
		c.JSON(http.StatusBadRequest, &jsonRsp)
		return
	}

	// Create Event
	dto, err := reposity.CreateItemFromDTO[models.DTO_Event, models.Event](dto)
	if err != nil {
		jsonRsp.Code = http.StatusInternalServerError
		jsonRsp.Message = err.Error()
		c.JSON(http.StatusInternalServerError, &jsonRsp)
		return
	}

	jsonRsp.Data = dto
	c.JSON(http.StatusCreated, &jsonRsp)
}

// ReadEvent	 godoc
// @Summary      Get single event by id
// @Description  Returns the event whose ID value matches the id.
// @Tags         events
// @Produce      json
// @Param        id  path  string  true  "Search event by id"
// @Success      200   {object}  models.JsonDTORsp[models.DTO_Event]
// @Router       /events/{id} [get]
// @Security		BearerAuth
func ReadEvent(c *gin.Context) {

	jsonRsp := models.NewJsonDTORsp[models.DTO_Event]()

	dto, err := reposity.ReadItemByIDIntoDTO[models.DTO_Event, models.Event](c.Param("id"))
	if err != nil {
		jsonRsp.Code = http.StatusNotFound
		jsonRsp.Message = err.Error()
		c.JSON(http.StatusNotFound, &jsonRsp)
		return
	}
	jsonRsp.Data = dto

	c.JSON(http.StatusOK, &jsonRsp)
}
```