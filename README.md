# Workshop Temporal IO - Modernizando hacia Banca Digital
Esta es una guia/workshop de temporal.io desarrollando un caso especifico para un mejor entendimiento de temporal y por que es ideal para el desarrollo de workflows resilentes.

## Intro Temporal.io

* **Workflow**: Flujos de trabajo: funciones o métodos de objeto que son el punto de entrada y la base de su aplicación.
* **Activities**: Actividades: funciones o métodos de objeto que manejan lógica empresarial no determinista.
* **Workers**: Trabajadores: procesos que se ejecutan en máquinas físicas o virtuales que ejecutan código de flujo de trabajo y actividad.
* **Signals**: Señales: llamadas de solo escritura a flujos de trabajo que pueden actualizar los valores de las variables y el estado del flujo de trabajo.
* **Queryes**: Consultas: llamadas de solo lectura a flujos de trabajo que pueden recuperar los valores de retorno de la función y el estado del flujo de trabajo.
* **Task Queues**: Colas de tareas: un mecanismo de enrutamiento que permite el equilibrio de carga.

## Caso de uso
El caso de uso que tomaremos es un flujo simplificado de una transferencia electronica, donde aprovecharemos de entender algunos de los conceptos basicos de temporal.

![flujo TEF](flujo_workflow_workshop.png)

Como se puede ver en el diagrama, este representa una transferencia electronica simplificada donde se realiza en primera instancia una verificacion del cliente que quiere realizar la transferencia y luego se realiza un cargo a la cuenta de origen y un abono a la cuenta de destino.

## Iniciando modulo
Para iniciar el modulo debemos hacer un go mod init.
```sh
$ go mod init github.com/jclanas2019/temporal-io-workshop-2021
```
Ojo con el nombre del repositorio en caso de que hagas un fork.

## Agregar dependencias
Este workflow depende principalmente solo de 2 librerias `temporal` y `fiber` vamos a instalarlas en nuestro modulo antes de iniciar con los desarrollos.

```bash
$ go get go.temporal.io/sdk
$ go get github.com/gofiber/fiber/v2
```

## Componentes del workflow
Este Workflow esta compuesto por 3 componentes: `workflow`, `starter` y `worker`.

### Workflow
Este es el workflow y las actividades que se realizan en este, el codigo de este componente corresponde al diagrama presentado en la seccion de arriba.

### Starter
Este componente se encarga de disparar el inicio de un workflow, en este caso mediante una API REST.

### Worker(s)
Este componente es quien se encarga de ejecutar la logica y las actividades del workflow, como es el componente que se encarga de estas actividades es ideal que para flujos complejos este sea el componente mas escalable.

## Construyendo el workflow

### Workflow
Para construir el workflow iniciaremos desarrollando el workflow y las actividades, para esto en el repositorio ya hay 2 archivos creados previamente para estos 2 casos, `workflow/workflow.go` y `workflow/activities.go`.

```go
package workflow

import (
	"errors"
	"time"

	"go.temporal.io/sdk/log"
	"go.temporal.io/sdk/workflow"
)

type Transfer struct {
	Origin      string `json:"origin"`
	Destination string `json:"destination"`
	Amount      int    `json:"amount"`
}

func TransferWorkflow(ctx workflow.Context, transfer Transfer) error {
	activityOptions := workflow.ActivityOptions{
		ScheduleToCloseTimeout: time.Minute,
		StartToCloseTimeout:    time.Second * 15,
	}

	ctx = workflow.WithActivityOptions(ctx, activityOptions)
	logger := workflow.GetLogger(ctx)

	if err := verifyCustomer(ctx, logger, transfer); err != nil {
		notifyFailedTransfer(ctx, transfer)
		return err
	}

	if err := executeTransfer(ctx, logger, transfer); err != nil {
		notifyFailedTransfer(ctx, transfer)
		return err
	}

	notifySuccessfulTransfer(ctx, transfer)

	return nil
}

func verifyCustomer(ctx workflow.Context, logger log.Logger, transfer Transfer) error {
	getCustomerInfoExec := workflow.ExecuteActivity(ctx, GetCustomerDetails, transfer.Origin)
	isCustomerRiskyExec := workflow.ExecuteActivity(ctx, IsRiskyCustomer, transfer.Origin)

	var customerName string
	err := getCustomerInfoExec.Get(ctx, &customerName)
	if err != nil {
		logger.Error("Error obteniendo informacion de cliente", transfer.Origin)
		return err
	}

	var isRisky bool
	err = isCustomerRiskyExec.Get(ctx, &isRisky)
	if err != nil {
		logger.Error("Error Resolviendo riesgo de cliente", transfer.Origin)
		return err
	}

	if isRisky {
		logger.Error("Cliente", transfer.Origin, "es riesgoso")
		return err
	}

	logger.Info("Cliente ", customerName, "Numero de cuenta", transfer.Origin, "no es riesgoso")

	return nil
}

func executeTransfer(ctx workflow.Context, logger log.Logger, transfer Transfer) error {
	chargeAccountExec := workflow.ExecuteActivity(ctx, ChargeAccount, transfer.Origin, transfer.Amount)
	payToAccountExec := workflow.ExecuteActivity(ctx, PayToAccount, transfer.Destination, transfer.Amount)

	chargeErr := chargeAccountExec.Get(ctx, nil)
	paymentError := payToAccountExec.Get(ctx, nil)

	if chargeErr != nil && paymentError != nil {
		logger.Error("Cargo fallido", chargeErr)
		logger.Error("Abono fallido", paymentError)

		return errors.New(chargeErr.Error() + " | " + paymentError.Error())
	} else if chargeErr != nil {
		logger.Error("Cargo fallido", chargeErr)
		workflow.ExecuteActivity(ctx, RevertPayment, transfer.Destination, transfer.Amount)
		return chargeErr
	} else if paymentError != nil {
		logger.Error("Abono fallido", paymentError)
		workflow.ExecuteActivity(ctx, RevertCharge, transfer.Origin, transfer.Amount)
		return paymentError
	}

	return nil
}

func notifyFailedTransfer(ctx workflow.Context, transfer Transfer) {
	defer workflow.ExecuteActivity(ctx, NotifyFailedTransfer, transfer.Origin, transfer.Destination, transfer.Amount).Get(ctx, nil)
}

func notifySuccessfulTransfer(ctx workflow.Context, transfer Transfer) {
	defer workflow.ExecuteActivity(ctx, NotifySuccessfulTransfer, transfer.Origin, transfer.Destination, transfer.Amount).Get(ctx, nil)
}
```
```go
package workflow

import (
	"log"
	"time"
)

func GetCustomerDetails(accountNumber string) (string, error) {
	time.Sleep(time.Millisecond * 20)
	log.Println("Cuenta identificada")
	return "Cliente 1", nil
}

func IsRiskyCustomer(accountNumber string) (bool, error) {
	time.Sleep(time.Millisecond * 100)
	log.Println("Cliente no es riesgoso")
	return false, nil
}

func ChargeAccount(accountNumber string, amount int) error {
	time.Sleep(time.Millisecond * 30)
	log.Println("Cargando", amount, "a cuenta", accountNumber)
	return nil
}

func PayToAccount(accountNumber string, amount int) error {
	time.Sleep(time.Millisecond * 30)
	log.Println("Abonando", amount, "a cuenta", accountNumber)
	return nil
}

func RevertCharge(accountNumber string, amount int) error {
	time.Sleep(time.Millisecond * 30)
	log.Println("Reversando cargo de", amount, "a cuenta", accountNumber)
	return nil
}

func RevertPayment(accountNumber string, amount int) error {
	time.Sleep(time.Millisecond * 30)
	log.Println("Reversando abono de", amount, "a cuenta", accountNumber)
	return nil
}

func NotifyFailedTransfer(origin, destination string, amount int) error {
	log.Println("Transaccion fallida de", amount, "desde", origin, "hacia", destination)
	return nil
}

func NotifySuccessfulTransfer(origin, destination string, amount int) error {
	log.Println("Transaccion exitosa de", amount, "desde", origin, "hacia", destination)
	return nil
}
```
### 

### Starter
Para poder iniciar este workflow construiremos una simple API rest con [fiber](https://gofiber.io/), para eso previamente tenemos creado un archivo `starter/main.go`.

```go
package main

import (
	"log"

	"github.com/donreno/temporal-io-workshop-2021/workflow"
	"github.com/gofiber/fiber/v2"
	"github.com/gofiber/fiber/v2/middleware/compress"
	"github.com/gofiber/fiber/v2/middleware/logger"
	"go.temporal.io/sdk/client"
)

func main() {
	// Inicializa temporal client
	c, err := client.NewClient(client.Options{})
	if err != nil {
		log.Fatalln("Error al crear cliente", err)
	}

	defer c.Close()

	workflowOpts := client.StartWorkflowOptions{
		ID:        "transfer-workflow",
		TaskQueue: "transfer-workflow-queue",
	}

	// Inicia server
	app := fiber.New()
	app.Use(logger.New())
	app.Use(compress.New())

	app.Post("/transfer", func(ctx *fiber.Ctx) error {
		var transfer workflow.Transfer
		ctx.BodyParser(&transfer)

		exec, err := c.ExecuteWorkflow(ctx.Context(), workflowOpts, workflow.TransferWorkflow, transfer)
		if err != nil {
			log.Println("Error iniciando workflow", err)
			return ctx.Status(500).SendString("Error iniciando workflow")
		}

		log.Println("Workflow ID", exec.GetID(), "| Run ID", exec.GetRunID())

		if err = exec.Get(ctx.Context(), nil); err != nil {
			log.Println("Error obteniendo resultado de workflow", err)
			return ctx.Status(500).SendString("Error obteniendo resultado de workflow")
		}

		return ctx.Status(200).SendString("Transferencia realizada de forma exitosa!")
	})

	log.Fatal(app.Listen(":3000"))
}
```

### Worker
Finalmente apra que este workflow sea atendido necesitamos desarrollar nuestro worker, para el cual previamente tenemos creado el archivo `worker/main.go`

```go
package main

import (
	"log"

	"github.com/donreno/temporal-io-workshop-2021/workflow"
	"go.temporal.io/sdk/client"
	"go.temporal.io/sdk/worker"
)

func main() {
	c, err := client.NewClient(client.Options{})
	if err != nil {
		log.Fatalln("Error al crear cliente", err)
	}

	defer c.Close()

	w := worker.New(c, "transfer-workflow-queue", worker.Options{})

	w.RegisterWorkflow(workflow.TransferWorkflow)
	w.RegisterActivity(workflow.GetCustomerDetails)
	w.RegisterActivity(workflow.IsRiskyCustomer)
	w.RegisterActivity(workflow.ChargeAccount)
	w.RegisterActivity(workflow.PayToAccount)
	w.RegisterActivity(workflow.RevertCharge)
	w.RegisterActivity(workflow.RevertPayment)
	w.RegisterActivity(workflow.NotifyFailedTransfer)
	w.RegisterActivity(workflow.NotifySuccessfulTransfer)

	if err = w.Run(worker.InterruptCh()); err != nil {
		log.Fatalln("Error ejecutando worker", err)
	}
}
```

## Iniciar temporal
Para levantar temporal simplemente hay que utilizar el docker compose que se encuentra en este repo
```bash
$ docker-compose up
```
Para mas detalles sobre este compose revisar [https://github.com/temporalio/docker-compose](https://github.com/temporalio/docker-compose).

## Iniciando starter y worker
Para iniciar el starter
```bash
$ go run ./starter
```
Y para inicial el worker
```bash
$ go run ./worker
```

## Probar workflow
Ahora podemos probar realizando una peticion en nuestra API

```bash
$ curl --location --request POST 'http://localhost:3000/transfer' \
--header 'Content-Type: application/json' \
--data-raw '{
    "origin": "0987654321",
    "destination": "0123456789",
    "amount": 500
}'
```
Probar matando la instancia de worker y/o temporal y volviendo a levantar para observar comportamiento.

Finalmente se pueden ver los resultados de ejecucion del workflow en [http://localhost:8088](http://localhost:8088).

1. Referencias:
	* https://docs.temporal.io/docs/concepts/introduction
	* https://docs.temporal.io/docs/server/introduction