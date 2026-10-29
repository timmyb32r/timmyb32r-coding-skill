# Примеры архитектурных решений

Примеры на Go показывают локальные преобразования. Имена зависимостей условны,
фрагменты не являются готовым приложением. Переноси принцип на язык проекта,
сохраняя реальный контракт. Стороны «до» могут быть уместны при других требованиях.

## 1. Один экземпляр — одна реальная единица работы

Условие: по контракту worker создаётся для одного export и затем закрывается.

До — экземпляр моделирует произвольное множество:

```go
type ExportManager struct {
    exportsByID map[string]*Export
}

func (m *ExportManager) Write(exportID string, rows []Row) error {
    export := m.exportsByID[exportID]
    if export == nil {
        return errors.New("export is not started")
    }
    return export.Write(rows)
}
```

После — экземпляр соответствует своему lifecycle:

```go
type Exporter struct {
    exportID string
    writer   RowWriter
}

func (e *Exporter) Write(rows []Row) error {
    return e.writer.Write(rows)
}
```

Роутинг по exportID остаётся у компонента, который действительно управляет
несколькими заданиями. Если один экземпляр обязан обслуживать несколько export,
map оправдана; не удаляй её вопреки контракту.

## 2. Алгоритм возвращает результат, вызывающий слой связывает эффекты

Условие: расчёт счёта не требует промежуточных записей или транзакции с чтением.

До — расчёт знает про весь сервис и самостоятельно меняет хранилище:

```go
type Calculator struct {
    service *BillingService
}

func (c *Calculator) Process(orderID string) error {
    order, err := c.service.orders.Load(orderID)
    if err != nil {
        return err
    }
    invoice := calculateInvoice(order)
    return c.service.invoices.Save(invoice)
}
```

После — расчёт зависит от данных, orchestration отвечает за эффекты:

```go
func BuildInvoice(order Order) (Invoice, error) {
    if len(order.Items) == 0 {
        return Invoice{}, errors.New("order has no items")
    }
    return calculateInvoice(order), nil
}

func (s *BillingService) CreateInvoice(orderID string) error {
    order, err := s.orders.Load(orderID)
    if err != nil {
        return fmt.Errorf("load order: %w", err)
    }
    invoice, err := BuildInvoice(order)
    if err != nil {
        return fmt.Errorf("build invoice: %w", err)
    }
    if err := s.invoices.Save(invoice); err != nil {
        return fmt.Errorf("save invoice: %w", err)
    }
    return nil
}
```

Направление: service → calculation и service → storage. Calculation не знает
про service/storage. Для внешних lookup передавай узкую необходимую зависимость;
не загружай всё заранее, если задача требует потоковой обработки.

Если эффект необходим внутри алгоритма, явная операция вроде
`beforeSend(ctx, message) error` допустима. Её контракт должен определять,
что ошибка запрещает отправку, а успешный возврат подтверждает нужный этап.
Не заменяй прямой результат произвольной цепочкой обработчиков.

## 3. Фиксированная привязка вместо подмены общего current-объекта

Условие: ReportBuilder содержит неизменяемые зависимости. Один Job обрабатывает
один вызов; параллельное выполнение разных заданий допустимо.

До — скрытая привязка и восстановление глобального для экземпляра контекста:

```go
func (b *ReportBuilder) BuildFor(job *Job) (Report, error) {
    previousJob := b.currentJob
    b.currentJob = job
    defer func() { b.currentJob = previousJob }()
    return b.build()
}
```

После — контекст операции виден в сигнатуре:

```go
func (b *ReportBuilder) Build(job JobSnapshot) (Report, error) {
    sections, err := b.buildSections(job)
    if err != nil {
        return Report{}, fmt.Errorf("build sections: %w", err)
    }
    return Report{JobID: job.ID, Sections: sections}, nil
}
```

Владелец Job выдаёт snapshot с копиями изменяемых вложенных коллекций, если
строитель может их менять. Передача struct по значению сама по себе не копирует
содержимое map/slice. Если объекту нужна длительная сессия, создай отдельный
`BuildSession` с фиксированным Job вместо перепривязки общего builder.

## 4. Структура данных вместо обёрток без собственного контракта

Условие: результат отделён от внутреннего состояния; обёртки не валидируют
и не обеспечивают дополнительной семантики.

До:

```go
type ItemView struct { item Item }
func (v ItemView) Name() string { return v.item.Name }
func (v ItemView) Size() int64 { return v.item.Size }

type Results struct { items []ItemView }
func (r Results) Items() []ItemView { return r.items }
```

После:

```go
type Item struct {
    Name string
    Size int64
}

func BuildItems(input Input) ([]Item, error) {
    // Build and validate the result before returning it.
    return buildItems(input)
}
```

Для простых значений публичные поля достаточны. Сохрани объект с закрытым
состоянием, когда его методы действительно обеспечивают инварианты или управляют
эффектами. Не выставляй наружу изменяемую запись хранилища ради удаления getters.

## 5. Одно состояние вместо противоречивых флагов

Условие: заказ находится ровно в одном из перечисленных состояний.

До:

```go
type Order struct {
    Paid      bool
    Canceled  bool
    Completed bool
}
```

После:

```go
type OrderStatus string

const (
    OrderPending   OrderStatus = "pending"
    OrderPaid      OrderStatus = "paid"
    OrderCanceled  OrderStatus = "canceled"
    OrderCompleted OrderStatus = "completed"
)

type Order struct {
    Status OrderStatus
}
```

Переходы и их сохранение выполняет владелец заказа. Такое представление убирает
комбинации вроде canceled+completed, но само по себе не валидирует переходы.
Если завершённый заказ должен отдельно хранить исторический факт оплаты,
`PaidAt` — самостоятельные данные, а не лишний флаг. Не своди независимые оси
в гигантский enum.

## 6. Модули по ответственности, без пакета на каждую пересылку вызова

Условие: преобразование строк — внутренняя деталь единственного exporter,
а два формата имеют существенно разные алгоритмы кодирования.

До:

```text
export/
  manager.go              # holds current job and every dependency
  context/                # passes the manager to other packages
  writer/                 # delegates every call to stream.Writer
  stream/                 # owns the actual output resource
  rowconversion/          # only writer imports it
  persistence/            # calls manager to update current job
```

После:

```text
export/
  exporter.go             # coordinates one export
  exporter_funcs.go       # local calculations used by exporter
  row_encoder.go          # private row representation/conversion
  format.go               # contract consumed by exporter
  csv_encoder.go          # CSV implementation
  json_encoder.go         # JSON implementation
job_store/
  store.go                # persists job state; no dependency on exporter
```

Exporter получает нужный store и encoder. Форматные реализации не зависят от
job_store. Отдельный stream-пакет остаётся уместен, если у него действительно
есть независимый контракт или несколько потребителей. Этот пример не задаёт
обязательное дерево: сначала выясни существующие границы проекта.

## 7. Имена и обычный путь должны раскрывать смысл

До:

```go
func (m *Manager) Do(ctx context.Context, id, name string) (err error) {
    defer func() {
        if err != nil {
            m.err = err
        }
    }()
    if m.err == nil {
        err = m.run(ctx, id, name)
    } else {
        err = m.err
    }
    return
}
```

После — тот же принцип сохранения первой ошибки виден непосредственно:

```go
func (e *Exporter) Export(ctx context.Context, jobID string, fileName string) error {
    if e.firstError != nil {
        return e.firstError
    }
    err := e.exportFile(ctx, jobID, fileName)
    if err != nil {
        e.firstError = err
        return err
    }
    return nil
}
```

Пример предполагает последовательный доступ. Если контракт допускает общий
экземпляр в нескольких goroutine, сохрани необходимую синхронизацию.
Не вводи firstError, если задача не требует прекращать работу после ошибки.
