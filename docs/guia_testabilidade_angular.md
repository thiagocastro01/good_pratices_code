# Guia de Testabilidade para Aplicações Angular

A testabilidade é um pilar fundamental no desenvolvimento de software de alta qualidade. Em aplicações Angular, focar na testabilidade significa escrever componentes, serviços e módulos que são fáceis de testar de forma isolada, rápida e confiável. Isso é alcançado principalmente através do uso de Injeção de Dependência, Serviços e uma boa organização de ambientes.

## 1. Princípios de Testabilidade em Angular

### Single Responsibility Principle (SRP)

Cada componente, serviço ou módulo deve ter uma única razão para mudar. Isso facilita o teste de uma funcionalidade específica sem se preocupar com efeitos colaterais.

### Injeção de Dependência (DI)

É o mecanismo central do Angular. Em vez de criar dependências diretamente dentro de uma classe, elas são fornecidas externamente. Isso permite "trocar" as dependências por versões mockadas ou simuladas durante os testes.

### Abstração e Interfaces

Embora TypeScript não force interfaces em tempo de execução, definir interfaces para serviços ou modelos de dados pode melhorar a clareza e facilitar o mocking (simulação) em testes.

### Componentes Puros (Dumb Components)

Componentes de apresentação que recebem dados via `@Input()` e emitem eventos via `@Output()`, sem lógica de negócio complexa ou chamadas a serviços diretamente. Eles são mais fáceis de testar isoladamente.

## 2. Ferramentas Essenciais

- **Angular CLI**: Gerencia o projeto e os comandos de teste (`ng test`, `ng e2e`).
- **Jasmine e Karma**: Frameworks padrão para testes de unidade e integração no Angular.
- **Protractor / Cypress / Playwright**: Frameworks para testes End-to-End (E2E). O Playwright é uma excelente escolha moderna.
- **Mock Service Worker (MSW)**: Ferramenta poderosa para interceptar requisições HTTP e retornar respostas mockadas, tanto em testes unitários/de componente quanto em E2E.

## 3. Aplicando Testabilidade: Exemplos Práticos

### 3.1. Serviços Angular (Comunicação com API)

```ts
// src/app/services/transacoes.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../../environments/environment';

export interface Transacao {
  id: string;
  valor: number;
  data: string;
}

@Injectable({ providedIn: 'root' })
export class TransacoesService {
  private apiUrl = environment.apiUrl;

  constructor(private http: HttpClient) { }

  registrarTransacao(transacao: Transacao): Observable<any> {
    return this.http.post(`${this.apiUrl}/transacoes`, transacao);
  }

  obterTransacao(id: string): Observable<Transacao> {
    return this.http.get<Transacao>(`${this.apiUrl}/transacoes/${id}`);
  }
}
```

### 3.2. Testando um Componente que Usa um Serviço

```ts
// src/app/components/transacao-form/transacao-form.component.ts
import { Component } from '@angular/core';
import { TransacoesService, Transacao } from '../../services/transacoes.service';

@Component({
  selector: 'app-transacao-form',
  template: `
    <input [(ngModel)]="transacao.id" placeholder="ID da Transação">
    <input [(ngModel)]="transacao.valor" type="number" placeholder="Valor">
    <button (click)="submit()">Registrar</button>
    <p *ngIf="message">{{ message }}</p>
  `
})
export class TransacaoFormComponent {
  transacao: Transacao = { id: '', valor: 0, data: '' };
  message: string = '';

  constructor(private transacoesService: TransacoesService) { }

  submit() {
    this.transacoesService.registrarTransacao(this.transacao).subscribe({
      next: () => { this.message = 'Transação registrada com sucesso!'; },
      error: (err) => { this.message = 'Erro ao registrar transação.'; console.error(err); }
    });
  }
}
```

#### Teste de Unidade

```ts
// transacao-form.component.spec.ts
import { ComponentFixture, TestBed } from '@angular/core/testing';
import { FormsModule } from '@angular/forms';
import { of, throwError } from 'rxjs';
import { TransacaoFormComponent } from './transacao-form.component';
import { TransacoesService } from '../../services/transacoes.service';

describe('TransacaoFormComponent', () => {
  let component: TransacaoFormComponent;
  let fixture: ComponentFixture<TransacaoFormComponent>;
  let mockTransacoesService: jasmine.SpyObj<TransacoesService>;

  beforeEach(async () => {
    mockTransacoesService = jasmine.createSpyObj('TransacoesService', ['registrarTransacao']);

    await TestBed.configureTestingModule({
      imports: [FormsModule],
      declarations: [TransacaoFormComponent],
      providers: [
        { provide: TransacoesService, useValue: mockTransacoesService }
      ]
    }).compileComponents();
  });

  beforeEach(() => {
    fixture = TestBed.createComponent(TransacaoFormComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should display success message on successful registration', () => {
    mockTransacoesService.registrarTransacao.and.returnValue(of({}));

    component.transacao = { id: 'test-id', valor: 100, data: '' };
    component.submit();

    expect(mockTransacoesService.registrarTransacao).toHaveBeenCalledWith(component.transacao);
    expect(component.message).toBe('Transação registrada com sucesso!');
  });

  it('should display error message on failed registration', () => {
    mockTransacoesService.registrarTransacao.and.returnValue(throwError(() => new Error('API Error')));

    component.transacao = { id: 'error-id', valor: 50, data: '' };
    component.submit();

    expect(mockTransacoesService.registrarTransacao).toHaveBeenCalledWith(component.transacao);
    expect(component.message).toBe('Erro ao registrar transação.');
  });
});
```

### 3.3. Gerenciamento de Ambientes (environments/)

```ts
// src/environments/environment.test.ts
export const environment = {
  production: false,
  test: true,
  apiUrl: 'http://localhost:4200/api'
};
```

```yaml
# Pipeline Azure DevOps
- script: |
    npm install
    ng build --configuration=test --output-path dist/angular-app-test
  displayName: 'Build Angular App for Test Environment'

- script: |
    cd dist/angular-app-test
    npx http-server . --port 4200 &
  displayName: 'Serve Angular App'

- script: |
    npx playwright test e2e/specs/transacao.e2e-spec.ts
  displayName: 'Run Playwright E2E Tests'
```

### 3.4. Mock Service Worker (MSW)

#### Handlers

```ts
// src/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.post('*/transacoes', () => {
    return HttpResponse.json({ message: 'Transação mockada registrada!' }, { status: 200 });
  }),
  http.get('*/transacoes/:id', ({ params }) => {
    const { id } = params;
    if (id === 'mock-1') {
      return HttpResponse.json({ id: 'mock-1', valor: 150.75, data: new Date().toISOString() }, { status: 200 });
    }
    return HttpResponse.json({ message: 'Transação mockada não encontrada.' }, { status: 404 });
  })
];
```

#### Teste E2E com Playwright

```ts
// e2e/specs/transacao.e2e-spec.ts
import { test, expect } from '@playwright/test';
import { setupWorker } from 'msw/browser';
import { handlers } from '../../src/mocks/handlers';

let worker: any;

test.beforeAll(async () => {
  worker = setupWorker(...handlers);
  await worker.start({ onUnhandledRequest: 'bypass' });
});

test.afterAll(async () => {
  await worker.stop();
});

test.describe('Transacao Form with MSW', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('http://localhost:4200');
  });

  test('should submit transaction and show success from mock', async ({ page }) => {
    await page.locator('input[placeholder="ID da Transação"]').fill('msw-test-id');
    await page.locator('input[placeholder="Valor"]').fill('99.99');
    await page.locator('button:text("Registrar")').click();

    await expect(page.locator('p')).toHaveText('Transação mockada registrada com sucesso!');
  });
});
```

## 4. Boas Práticas para Testabilidade em Angular

- **Evite lógica complexa em componentes**: Mova a lógica de negócio para serviços.
- **Mantenha os componentes "burros"**: Foco em apresentação e delegação.
- **Use HttpClientTestingModule**: Ideal para testes de serviços.
- **Padronize seus ambientes**: Mantenha `environment.*.ts` atualizados.
- **Teste o contrato da API**: Com MSW ou Testcontainers para simulações confiáveis.

Ao seguir esses princípios e utilizar as ferramentas corretas, suas aplicações Angular serão significativamente mais robustas, fáceis de manter e, o mais importante, mais confiáveis.

