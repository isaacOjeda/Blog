### Introducción

En el desarrollo de aplicaciones web modernas, la reutilización y la mantenibilidad del código son cruciales para garantizar el éxito a largo plazo. Una de las formas de lograrlo es mediante la creación de componentes personalizados, que no solo encapsulan funcionalidad específica, sino que también sirven como una capa de abstracción sobre las bibliotecas externas. Este enfoque permite minimizar la dependencia directa de terceros, reduciendo el impacto de futuros cambios en estas bibliotecas.

En este artículo, exploraremos cómo crear un componente personalizado en Angular que actúe como una abstracción sobre un componente de Kendo UI. También abordaremos la importancia de la abstracción en sistemas de largo plazo y cómo las dependencias externas pueden convertirse en un problema en sistemas grandes.

### Creación de Componentes Personalizados

#### Creación del Date Picker

Imaginemos que estás utilizando Kendo UI en tu proyecto Angular para manejar componentes UI como el Date Picker. Sin embargo, has notado que cualquier actualización en Kendo UI podría forzarte a cambiar decenas o incluso cientos de archivos en tu código base. Para mitigar este riesgo, puedes crear un componente personalizado que encapsule la lógica del Date Picker, permitiéndote modificar la implementación subyacente sin afectar el resto de tu aplicación.

Aquí te mostramos cómo hacerlo:

```typescript
import { Component, EventEmitter, forwardRef, Input, Output } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';

@Component({
  selector: 'custom-date-input',
  template: `
    <kendo-datepicker [format]="format" [value]="value" (valueChange)="onValueChange($event)" [disabled]="disabled">
    </kendo-datepicker>
  `,
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => DateInputComponent),
      multi: true,
    }
  ]
})
export class DateInputComponent implements ControlValueAccessor {
  @Input()
  public value: Date | null = null;

  @Input()
  public format: string = 'dd/MM/yyyy';

  @Output()
  public valueChange: EventEmitter<Date> = new EventEmitter<Date>();

  public disabled: boolean = false;

  private onChange = (value: any) => { };
  private onTouched = () => { };

  writeValue(value: any): void {
    this.value = value;
  }

  registerOnChange(fn: any): void {
    this.onChange = fn;
  }

  registerOnTouched(fn: any): void {
    this.onTouched = fn;
  }

  setDisabledState?(isDisabled: boolean): void {
    this.disabled = isDisabled;
  }

  onValueChange(value: Date): void {
    this.value = value;
    this.onChange(value);
    this.onTouched();
    this.valueChange.emit(value);
  }
}

```

Este componente personalizado `DateInputComponent` actúa como una abstracción sobre el componente `kendo-datepicker`, permitiendo el binding bidireccional de su valor y emitiendo eventos cuando el valor cambia. Esto permite que tu aplicación utilice un Date Picker sin estar acoplada directamente a la implementación de Kendo.

### Explicación del Código del `DateInputComponent`

El `DateInputComponent` es un componente Angular personalizado que encapsula un `kendo-datepicker`, proporcionando una interfaz consistente y reutilizable para manejar entradas de fecha en tu aplicación. A continuación, desglosamos cada parte clave del componente:

```typescript
import { Component, EventEmitter, forwardRef, Input, Output } from '@angular/core';
import { ControlValueAccessor, NG_VALUE_ACCESSOR } from '@angular/forms';
```

#### Importaciones y Configuración Inicial

- **Angular Core Imports**: Importamos las funcionalidades principales de Angular como `Component`, `Input`, `Output`, y `EventEmitter`.
- **ControlValueAccessor**: Esta interfaz permite que nuestro componente actúe como un puente entre el valor de un formulario y la vista del componente, permitiendo la integración con formularios reactivos y basados en plantillas.
- **NG_VALUE_ACCESSOR**: Utilizamos este token para registrar nuestro componente como un proveedor de `ControlValueAccessor`, lo que permite que Angular lo reconozca como un input de formulario.

#### Decorador @Component

```typescript
@Component({
  selector: 'darwin-date-input',
  template: `
    <kendo-datepicker [format]="format" [value]="value" (valueChange)="onValueChange($event)" [disabled]="disabled">
    </kendo-datepicker>
  `,
  styleUrls: ['./date-input.component.css'],
  providers: [
    {
      provide: NG_VALUE_ACCESSOR,
      useExisting: forwardRef(() => DateInputComponent),
      multi: true,
    }
  ]
})
```

- **selector**: Define el nombre del elemento HTML que representará este componente en la aplicación.
- **template**: El componente encapsula un `kendo-datepicker`, pasando las propiedades como `format`, `value`, y `disabled`. También maneja el evento `valueChange` para actualizar el valor del componente.
- **providers**: Registra el componente como un `ControlValueAccessor`, permitiendo que funcione en formularios de Angular.

#### Propiedades y Constructor

```typescript
export class DateInputComponent implements BaseInput<Date>, ControlValueAccessor {
  @Input()
  public value: Date | null;
  @Input()
  public format: string = 'dd/MM/yyyy';

  @Output()
  public valueChange: EventEmitter<Date>;
  public disabled: boolean = false;

  private onChange = (value: any) => { };
  private onTouched = () => { };

  constructor() {
    this.value = null;
    this.valueChange = new EventEmitter<Date>();
  }

```

- **@Input() value**: Este es el valor del `kendo-datepicker`, que se enlaza bidireccionalmente con el formulario.
- **@Input() format**: Permite configurar el formato de fecha desde el exterior del componente. Si no se proporciona un formato, utiliza uno por defecto.
- **@Output() valueChange**: Este evento se emite cuando el valor de la fecha cambia, permitiendo a otros componentes reaccionar a los cambios de valor.
- **disabled**: Controla si el input de fecha está habilitado o deshabilitado.
- **onChange** y **onTouched**: Estas funciones son definidas para cumplir con la interfaz `ControlValueAccessor`, permitiendo que Angular controle el estado y los cambios del input.

#### Métodos Implementados

```typescript
writeValue(obj: any): void {
  this.value = obj;
}

registerOnChange(fn: any): void {
  this.onChange = fn;
}

registerOnTouched(fn: any): void {
  this.onTouched = fn;
}

setDisabledState?(isDisabled: boolean): void {
  this.disabled = isDisabled;
}

onValueChange(value: Date): void {
  this.value = value;
  this.onChange(value);
  this.onTouched();
}
```

- **writeValue(obj: any)**: Este método se invoca cuando Angular necesita actualizar el valor del componente desde el formulario. Simplemente asigna el valor recibido a la propiedad `value`.
- **registerOnChange(fn: any)** y **registerOnTouched(fn: any)**: Angular pasa funciones a estos métodos para manejar los cambios en el componente y cuando el usuario interactúa con él. Guardamos estas funciones para llamarlas posteriormente.
- **setDisabledState(isDisabled: boolean)**: Permite a Angular habilitar o deshabilitar el componente según sea necesario.
- **onValueChange(value: Date)**: Este método se invoca cuando el usuario selecciona una fecha nueva. Actualiza el valor del componente y notifica a Angular sobre el cambio, garantizando que la vista y el modelo del formulario estén sincronizados.

### Resumen

El `DateInputComponent` es un ejemplo de cómo encapsular la funcionalidad de componentes de terceros, como Kendo UI, dentro de tu propia interfaz abstracta. Este enfoque no solo te proporciona mayor flexibilidad y control sobre tu código, sino que también te protege contra cambios en las bibliotecas externas, minimizando el impacto en tu base de código cuando necesitas actualizar o reemplazar esas dependencias.

#### Unit Testing

Es fundamental garantizar que tu componente personalizado funcione correctamente en todas las situaciones posibles. Para ello, puedes crear pruebas unitarias que validen su comportamiento:

```typescript
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { DateInputComponent } from './date-input.component';
import { FormsModule, ReactiveFormsModule } from '@angular/forms';
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
import { DateInputsModule } from '@progress/kendo-angular-dateinputs';

describe('DateInputComponent', () => {
  let component: DateInputComponent;
  let fixture: ComponentFixture<DateInputComponent>;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [DateInputComponent],
      imports: [FormsModule, ReactiveFormsModule, BrowserAnimationsModule, DateInputsModule]
    }).compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(DateInputComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeTruthy();
  });

  it('should emit valueChange event when value changes', () => {
    spyOn(component.valueChange, 'emit');
    const newValue = new Date('2024-01-01');
    component.onValueChange(newValue);
    expect(component.valueChange.emit).toHaveBeenCalledWith(newValue);
  });

  it('should reflect the value in the template', () => {
    const newValue = new Date('2024-01-01');
    component.writeValue(newValue);
    fixture.detectChanges();
    const datePickerElement: HTMLElement = fixture.nativeElement.querySelector('kendo-datepicker');
    expect(datePickerElement.getAttribute('ng-reflect-value')).toBe('2024-01-01T00:00:00.000Z');
  });
});

```

Estas pruebas aseguran que el componente funcione correctamente en términos de creación, emisión de eventos, y sincronización del valor con la plantilla.

### Uso del Componente Personalizado

Una vez que has creado tu componente `DateInputComponent`, puedes usarlo en formularios tanto reactivos como basados en plantillas. A continuación, se muestra cómo integrar este componente en ambos enfoques.

> 💡 **Nota:** Este componente no se encuentra en ningún módulo, será necesario que hagas tu módulo de componentes y luego utilizarlo en algún otro módulo o importar todos tus componentes directamente en el módulo donde lo necesites.
> 

#### Uso con Reactive Forms

Los formularios reactivos en Angular son ideales para manejar formularios complejos, donde se necesita un mayor control sobre la validación y la lógica del formulario. Aquí te muestro cómo utilizar el componente `DateInputComponent` en un formulario reactivo:

```typescript
import { Component, OnInit } from '@angular/core';
import { FormBuilder, FormGroup } from '@angular/forms';

@Component({
  selector: 'app-reactive-form-example',
  template: `
    <form [formGroup]="form">
      <label for="date">Select Date:</label>
      <custom-date-input formControlName="date" #dateInput></custom-date-input>
    </form>
    <p>Selected Date: {{dateInput.value | date: 'dd/MM/yyyy'}}</p>
  `
})
export class ReactiveFormExampleComponent implements OnInit {
  form!: FormGroup;

  constructor(private fb: FormBuilder) {}

  ngOnInit(): void {
    this.form = this.fb.group({
      date: [new Date()]
    });
  }
}

```

En este ejemplo:

1. Se utiliza `FormBuilder` para crear un formulario reactivo con un control de fecha (`date`).
2. El componente `DateInputComponent` se integra con el formulario utilizando `formControlName`.
3. La fecha seleccionada se muestra debajo del formulario utilizando la propiedad `value` del control de fecha.

#### Uso con Template-Driven Forms

Los formularios basados en plantillas son más adecuados para formularios más simples, donde el control directo de la lógica y la validación del formulario no es tan crucial. Aquí se muestra cómo usar el componente `DateInputComponent` en un formulario basado en plantillas:

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-template-driven-form-example',
  template: `
    <form #form="ngForm">
      <label for="date">Select Date:</label>
      <custom-date-input name="date" [(ngModel)]="date"></custom-date-input>
    </form>
    <p>Selected Date: {{ date | date }}</p>
  `
})
export class TemplateDrivenFormExampleComponent {
  date: Date = new Date();
}

```

En este ejemplo:

1. El componente `DateInputComponent` se utiliza dentro de un formulario basado en plantillas.
2. Se usa `[(ngModel)]` para crear un binding bidireccional entre el valor del componente y la propiedad `date`.
3. La fecha seleccionada se muestra debajo del formulario utilizando el valor de `date`.

### Conclusión

La abstracción es una herramienta poderosa en el desarrollo de software, especialmente en proyectos de larga duración. Al encapsular componentes de bibliotecas externas en componentes personalizados, puedes proteger tu código de los cambios en estas dependencias, mejorando así la mantenibilidad y reduciendo el riesgo de errores en el futuro.

En sistemas grandes, la dependencia directa de bibliotecas externas puede llevar a problemas de actualización y mantenimiento, ya que los cambios en esas bibliotecas pueden requerir modificaciones extensivas en el código. Al crear una capa de abstracción, puedes aislar estas dependencias y asegurar que las futuras actualizaciones de las bibliotecas tengan un impacto mínimo en tu aplicación.

Este enfoque no solo facilita la gestión del código a largo plazo, sino que también mejora la flexibilidad y la capacidad de adaptación de tu aplicación a nuevas tecnologías y necesidades emergentes. La clave es encontrar un equilibrio entre reutilización y abstracción para mantener un código limpio, modular y fácil de mantener.