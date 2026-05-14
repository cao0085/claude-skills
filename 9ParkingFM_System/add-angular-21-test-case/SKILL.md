---
name: add-angular-21-test-case
description: "Scaffold the index.ts component for an Angular 21 feature. Use this skill whenever: user runs /add-angular21-index, asks to initialize or fill in an index component, or wants to wire up an api.ts into a component with signals, CRUD methods, and optional mock mode."
---


##

```js

// 驗證正規表達
describe('routeValidator', () => {
  let control: FormControl;

  beforeEach(() => {
    control = new FormControl();
  });

  it.each(['my-route-123'])('允許值: %s', (value) => {
    control.setValue(value);
    expect(routeValidator(control)).toBeNull();
  });

  it.each(['', null, undefined,'ABC', 'my route', 'my_route', 'my.route', 'my@route'])('拒絕值: %s', (value) => {
    control.setValue(value);
    expect(routeValidator(control)).toEqual({ invalidRoute: true });
  });
});
```

```js
describe('MenuManagement - Component', () => {
  let component: MenuManagement;

  const mockResource = {
    value: signal<MenuDetail | undefined>(undefined),
    reload: vi.fn(),
  };

  const mockApi: Partial<MenuManagementApi> = {
    menuDetailResource: vi.fn().mockReturnValue(mockResource),
    getMenus:           vi.fn().mockReturnValue(of([])),
    getActions:         vi.fn().mockReturnValue(of([])),
    reorderMenus:       vi.fn().mockReturnValue(of({})),
  };

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [MenuManagement],
      providers: [
        provideNoopAnimations(),
        { provide: NzMessageService, useValue: { success: vi.fn(), error: vi.fn() } },
        { provide: NzModalService,   useValue: {} },
      ],
    })
    .overrideComponent(MenuManagement, {
      set: { providers: [{ provide: MenuManagementApi, useValue: mockApi }] },
    })
    .compileComponents();

    const fixture = TestBed.createComponent(MenuManagement);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  describe('onNodeClick', () => {
    it.each([
      { desc: '有 key 的節點',    key: 'menu-abc', expected: 'menu-abc' },
      { desc: '無 key 的節點',    key: undefined,  expected: null       },
    ])('$desc → selectedMenuId 更新', ({ key, expected }) => {
      component.onNodeClick({ node: key ? { key } : null } as unknown as NzFormatEmitEvent);
      expect(component.selectedMenuId()).toBe(expected);
    });
  });

  describe('beforeDrop', () => {
    function arg(dragParent: string | null, dropParent: string | null) {
      return {
        dragNode: { parentNode: dragParent != null ? { key: dragParent } : null },
        node:     { parentNode: dropParent != null ? { key: dropParent } : null },
      } as unknown as NzFormatBeforeDropEvent;
    }

    it.each([
      { desc: '根層調整排序', drag: null,  drop: null,  expected: true  },
      { desc: '同子層調整排序',     drag: 'p1',  drop: 'p1',  expected: true  },
      { desc: '不可跨層調整排序',     drag: 'p1',  drop: 'p2',  expected: false },
    ])('$desc', ({ drag, drop, expected }) => {
      let result: boolean | undefined;
      component.beforeDrop(arg(drag, drop)).subscribe(v => result = v);
      expect(result).toBe(expected);
    });
  });
});
```