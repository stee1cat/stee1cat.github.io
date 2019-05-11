---
layout: post
title: Компонент Delphi для автозаполнения в строке ввода
tags: [autocomplete, Delphi, автозаполнение, компонент]
---

![]({{ site.baseurl }}/images/autocomplete.png)

Наступил май, а это значит, что у студентов приближаются сроки сдачи курсовых работ. И поэтому недавно попросили меня помочь в написании такой работы. Недолго думая я согласился, хотя опыта в программировании на Delphi у меня немного. Задача состояла в том, чтобы «расширить возможности TEdit, так чтобы он мог работать как T9, например». По техническому заданию, становиться понятно, что нужно что-то вроде, предложения для вставки слов по уже введённым символам, т. е. обычное автозаполнение. По началу у меня ещё были мысли о попытке исправления неправильно введённого слова, но после разговора с «заказчиком» стало понятно, что это необязательно. Так как я не открывал [IDE](http://ru.wikipedia.org/wiki/Delphi_(%D1%81%D1%80%D0%B5%D0%B4%D0%B0_%D1%80%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B8)) от [Borland](http://www.borland.com/) (или сейчас уже от [Embarcadero Technologies](http://www.embarcadero.com/)) больше года, то для того, что бы не затягивать сроки, изобретение велосипеда было отложено на потом.

Мною был найден и переработан исходный код компонента указанный в источниках к статье. TComboBox я заменил на поле TEdit, добавил загрузку словаря из файла, существенно изменил поиск по подстроке. Автозаполнение срабатывает при наборе слова из любого положения курсора в строке ввода. Начальной строкой для поиска по словарю служит строка слева от курсора и до первого пробела справа от начала строки ввода. Из этих ограничений невозможен поиск словосочетаний, даже если они имеются в словаре. На будущее, есть идея реализовать возможность добавления слова отсутствующего в словаре. Весь код приведён ниже. Компилировал в Delphi XE, вроде как никаких особенностей этой версии среды не использовал, поэтому должно работать и в более ранних.

**20/05/2012 — Обновлено:** Добавлена вставка нового слова в словарь через контекстное меню поля ввода. Убраны мелкие ошибки, исправлен кривой поиск. Всё же можно сделать поиск и по словосочетаниям, но уже стимула нет, т. к. шоколадка получена, а девушка довольна результатом :).

```pascal
unit AutoComplete;
 
interface
 
uses
  SysUtils, Windows, Messages, Classes, Controls, StdCtrls, Forms, Dialogs, Menus;
 
// Класс выпадающего меню автозавершения.
type
  TDropDown = class(TListBox)
  protected
    procedure Sort;
    procedure CreateParams(var Params: TCreateParams);
      override;
    procedure OnActivateApp(var M: TMessage);
      message WM_ACTIVATEAPP;
    procedure OnLButtonUp(var M: TWMLButtonUp);
      message WM_LBUTTONUP;
    procedure OnMouseOver(var M: TWMMouseMove);
      message WM_MOUSEMOVE;
end;
 
// Класс контекстного меню поля ввода.
type
  TMenu = class(TPopupMenu)
    AddWord, Item, Cut, Copy, Delete, Paste, SelectAll: TMenuItem;
  protected
    procedure AddWordClick(Sender: TObject);
    procedure CutClick(Sender: TObject);
    procedure CopyClick(Sender: TObject);
    procedure PasteClick(Sender: TObject);
    procedure DeleteClick(Sender: TObject);
    procedure SelectAllClick(Sender: TObject);
    procedure OnPopupMenu(Sender: TObject);
  public
    constructor Create(AOwner: TComponent);
      override;
end;
 
// Класс поля ввода текста.
type
  TAutoComplete = class(TEdit)
  private
    InsertText: boolean;                          // Переменная для отслеживания
                                                  // вставки текста в поле
                                                  // TAutoComplete.
    Previous: string;                             // Предыдущее значение поля.
    DictionaryFile: string;                       // Файл словаря.
    FDropped: boolean;                            // True когда показан список.
    FDictionary: TStrings;
    FOldFormWndProc, FNewFormWndProc: Pointer;    // Используется для подмены
                                                  // оконной процедуры
                                                  // родительской формы.
    FParentFormWnd: hWnd;                         // Handle родительской формы.
    procedure AddWord;
    procedure LoadDictionary;
    procedure FindStrings(const AText: string);
    procedure ParentFormWndProc(var M: TMessage); // Оконная процедура для
                                                  // подмены процедуры
                                                  // родительской формы.
    function CompareString(const EditText, DictionaryString: string): boolean;
    function RightTrim(const EditText: string): string;
    function LeftTrim(const EditText: string): string;
  protected
    FMenu: TMenu;                                 // Контекстное меню поля ввода.
    FDropDown: TDropDown;                         // Выпадающее окно автозавершения.
    procedure KeyDown(var Key: Word; Shift: TShiftState);
      override;
    procedure SetParent(AParent: TWinControl);
      override;
    procedure OnChange(var M: TWMCommand);
      message CN_COMMAND;
    procedure OutOfFocus(var M: TCMExit);
      message CM_EXIT;
  public
    constructor Create(AOwner: TComponent);
      override;
    destructor Destroy;
      override;
    procedure ShowList;
    procedure HideList(InsertSelection: boolean);
  published
 
end;
 
procedure Register;
 
implementation
 
  procedure Register;
  begin
    RegisterComponents('Samples', [TAutoComplete]);
  end;
 
  constructor TAutoComplete.Create(AOwner: TComponent);
  var i: integer;
  begin
    inherited Create(AOwner);
    DictionaryFile := 'dictionary.txt';
    Previous := '';
    InsertText := false;
    FDropped := false;
    FParentFormWnd := 0;
    // В Design-Time список автозавершения не нужен,
    // поэтому создаётся только в Run-Time.
    if not (csDesigning in ComponentState) then
      begin
        FMenu := TMenu.Create(self);
        PopupMenu := FMenu;
        FDropDown := TDropDown.Create(self);
        LoadDictionary;
      end;
  end;
 
  destructor TAutoComplete.Destroy;
  begin
    if not (csDesigning in ComponentState) then
      begin
        // Удаляем все динамически созданные объекты.
        FMenu.Free;
        FDropDown.Free;
        FDictionary.Free;
      end;
    inherited Destroy;
  end;
 
  // Вызывается из метода FindStrings для каждого элемента списка FDictionary.
  // Используется для сравнения строк.
  function TAutoComplete.CompareString(const EditText, DictionaryString: string): boolean;
  var i, l: integer;
      not_eqvals: boolean;
  begin
    l := length(EditText);
    // Выходим при длине слова равному нулю.
    if l = 0 then
      Result := false
    else
      begin
        // Если длина строки из поля TAutoComplete превышает длину строки из словаря,
        // то тогда выходим.
        if l > length(DictionaryString) then
          Result := false
        else
          begin
            not_eqvals := false;
            // Последовательно проходим по всем символам слова из поля ввода.
            for i := 1 to l do
              begin
                if AnsiCompareText(EditText[i], DictionaryString[i]) <> 0 then
                  begin
                    not_eqvals := true;
                    break;
                  end;
              end;
            if not_eqvals = true then
              Result := false
            else
              Result := true;
          end;
      end;
  end;
 
  // Обрезает строку слева до первого с права пробела. Если в строке нет пробелов,
  // то будет возвращена строка целиком.
  function TAutoComplete.LeftTrim(const EditText: string): string;
  var i: integer;
      trim: string;
  begin
    trim := '';
    for i := length(EditText) downto 1 do
      begin
        if AnsiCompareText(EditText[i], ' ') = 0 then
          break;
        trim := EditText[i] + trim;
      end;
    Result := trim;
  end;
 
  // Функция обрезает строку справа до последнего встретившегося пробела с начала
  // строки включительно. При отсутствии пробела (т. е. строка не содержит пробелов),
  // возвращаем пустую строку.
  function TAutoComplete.RightTrim(const EditText: string): string;
  var trim: string;
  begin
    if length(EditText) = length(LeftTrim(EditText)) then
      begin
        Result := '';
      end
    else
      begin
        trim := EditText;  // Временная переменная, доступная для изменения.
        delete(trim, length(EditText) - length(LeftTrim(EditText)) + 1, length(EditText));
        Result := trim;
      end;
  end;
 
  // Процедура сохранения слова в файле словаря.
  procedure TAutoComplete.AddWord;
  var i: integer;
      word_exists: boolean;
      NewWord: string;
  begin
    word_exists := false;
    // Добавляем строку в словарь, только при длине > 0
    // и при отсутствии пробелов.
    if length(SelText) > 0 then
      begin
        // Обрезаем пробелы слева и справа.
        NewWord := trim(SelText);
        // Если в полученной строке нет пробелов (необходимо добавть только
        // слово, т. к. поиск работает по отдельным словам), то добавляем.
        if (pos(' ', NewWord) = 0) then
          begin
            // Перебираем строки из списка FDictionary и сравниваем с строкой
            // претендующей на добавление. При первом совпадении прекращаем поиск.
            for i := 0 to FDictionary.Count -1 do
                if AnsiCompareText(NewWord, FDictionary.Strings[i]) = 0 then
                  begin
                    word_exists := true;
                    break;
                  end;
            // Если страка не найдена в словаре, то добавляем её.
            if word_exists = false then
              begin
                FDictionary.Add(AnsiLowerCase(NewWord));
                FDictionary.SaveToFile(DictionaryFile);
                ShowMessage('Слово добавлено!');
              end
            else
              ShowMessage('Слово уже есть в словаре!');
          end
        else
          ShowMessage('В словарь можно добавлять только отдельные слова!');
      end
  end;
 
  // Процедура загрузки словаря из файла.
  procedure TAutoComplete.LoadDictionary;
  begin
    FDictionary := TStringList.Create();
    if (fileexists(DictionaryFile)) then
      FDictionary.LoadFromFile(DictionaryFile)
    else
      ShowMessage('Словарь не загружен!');
  end;
 
  procedure TAutoComplete.HideList(InsertSelection: boolean);
  var i: integer;
      lString, rString: string;
  begin
    // Скрываем список автозавершения.
    ShowWindow(FDropDown.Handle, SW_HIDE);
    // Если InsertSelection = true, то помещаем в поле редактирования TListBox
    // выбранный элемент из списка автозавершения.
    if InsertSelection then
      begin
        i := FDropDown.ItemIndex;
        if i <> -1 then
          begin
            lString := Text;
            rString := Text;
            // Строка слева от курсора.
            delete(lString, SelStart + 1, length(Text) - SelStart);
            // Строка справа от курсора.
            delete(rString, 1, length(lString));
            // Временно блокируем показ окна автозаполения, т. к. оно срабатывает
            // на событие OnChange.
            InsertText := true;
            Text := RightTrim(lString) + FDropDown.Items[i] + rString;
            // Разрешаем.
            InsertText := false;
            // Устанавливаем курсор в строку поля ввода.
            SelStart := length(RightTrim(lString) + FDropDown.Items[i]);
          end;
      end;
    FDropped := false;
  end;
 
  procedure TAutoComplete.ShowList;
  var p: TPoint;
      result: integer;
  begin
    // Если в поле ввода TAutoComplete пусто, то прячем список автозавершения.
    if Text = '' then
      begin
        HideList(false);
        exit;
      end
    else
      begin
        FindStrings(Text);
        // Заполняем список автозаверения вариантами, соответствующими введенному
        // тексту.
        result := FDropDown.Items.Count;
        // Если для введенного текста подходящих вариантов нет, прячем список
        // автозавершения.
        if result = 0 then
          begin
            HideList(false);
            exit;
          end;
        // Задаем высоту окна автозавершения, чтобы в нем помещалось не более
        // пяти строк. При большем количестве вариантов, будет показана
        // вертикальная полоса прокрутки.
        if result > 5 then
          result := 5;
        FDropped := true;
        // Показываем окно автозавершения под TAutoComplete.
        p.x := 0;
        p.y := height - 1;
        p := ClientToScreen(p);
        SetWindowPos(FDropDown.Handle, HWND_TOPMOST, p.x, p.y,
          width - GetSystemMetrics(SM_CXVSCROLL),
          result * FDropDown.ItemHeight + 2, SWP_SHOWWINDOW);
      end;
  end;
 
  procedure TAutoComplete.KeyDown(var Key: Word; Shift: TShiftState);
  var M: TWMKeyDown;
  begin
    case Key of
      // По нажатию Esc скрываем список автозавершения.
      VK_ESCAPE:
        HideList(false);
      // Нажатия стрелок Up/Down, PgUp/PgDn передаём окну автозавершения, чтобы
      // перемещаться по списку найденных вариантов.
      VK_UP, VK_DOWN, VK_NEXT, VK_PRIOR:
        begin
          FillChar(M, SizeOf(M), 0);
          M.Msg := WM_KEYDOWN;
          M.CharCode := key;
          SendMessage(FDropDown.Handle, TMessage(M).Msg, TMessage(M).wParam, TMessage(M).lParam);
          FillChar(M, SizeOf(M), 0);
          M.Msg := WM_KEYUP;
          M.CharCode := Key;
          SendMessage(FDropDown.Handle, TMessage(M).Msg, TMessage(M).wParam, TMessage(M).lParam);
          // Скрываем от ОС факт нажатия Up/Down и PgUp/PgDn.
          Key := 0;
        end;
      // По нажатию Enter закрываем окно автозавершения и помещаем в поле
      // TAutoComplete выбранный элемент из списка автозавершения.
      VK_RETURN:
        HideList(true);
    else
      inherited KeyDown(Key, Shift);
    end;
  end;
 
  procedure TAutoComplete.OnChange(var M: TWMCommand);
  begin
    // При изменении текста в поле TAutoComplete показываем список автозавершения.
    // Так же проводим проверку на случай вставки слова из списка, тогда окно не
    // будет показано.
    if (M.NotifyCode = EN_CHANGE) and (Previous <> Text) and (not InsertText) then
      begin
        Previous := Text;
        ShowList;
      end
    else
      inherited;
  end;
 
  procedure TAutoComplete.SetParent(AParent: TWinControl);
  var ParentForm: TCustomForm;
  begin
    // Если компонент переносится с одной формы на другую,
    // то возвращаем форме её родную оконную процедуру.
    if not (csDesigning in ComponentState) and (FParentFormWnd <> 0) then
      SetWindowLong(FParentFormWnd, GWL_WNDPROC, Integer(FOldFormWndProc));
    inherited SetParent(AParent);
    // Подменяем оконную процедуру родительской формы. Делаем это только в Run-Time,
    // т.к. в Design-Time список автозавершения не создается.
    if not (csDesigning in ComponentState) then
      begin
        ParentForm := GetParentForm(self);
        if Assigned(ParentForm) then
          begin
            FParentFormWnd := ParentForm.Handle;
            FNewFormWndProc := MakeObjectInstance(ParentFormWndProc);
            FOldFormWndProc := Pointer(GetWindowLong(FParentFormWnd, GWL_WNDPROC));
            SetWindowLong(FParentFormWnd, GWL_WNDPROC, Integer(FNewFormWndProc));
          end;
      end;
  end;
 
  // Процедура подменяющая оконную процедуру родительской формы. Необходима для
  // отслеживания изменения положения родительского окна.
  procedure TAutoComplete.ParentFormWndProc(var M: TMessage);
    procedure Default;
    begin
      with M do
        Result := CallWindowProc(FOldFormWndProc, FParentFormWnd, Msg, wParam, lParam);
    end;
  begin
    case M.Msg of
      // При изменении положения родительской формы прячем окно автозавершения.
      WM_WINDOWPOSCHANGING, WM_WINDOWPOSCHANGED:
        HideList(false);
    end;
    Default;
  end;
 
  // Метод заполняет список вариантов автозавершения. Варианты берутся из FDictionary.
  // В параметре AText передается введенный в поле TAutoComplete текст.
  procedure TAutoComplete.FindStrings(const AText: string);
  var i: integer;
      FindString, lSel: string;
      FDropDownSorted: TStringList;
  begin
    // Если курсор находится не в конце строки, то находим строку для поиска.
    if (SelStart) <> length(AText) then
      begin
        lSel := AText;
        FindString := AText;
        // Получаем строку слева от курсора.
        delete(lSel, SelStart + 1, length(AText) - SelStart);
        FindString := lSel;
        // Получаем строку от первого пробела слева от курсора до самого курсора.
        delete(FindString, 0, length(RightTrim(lSel)));
        FindString := LeftTrim(FindString);
      end
    else
        FindString := LeftTrim(AText);
    FDropDown.Items.Clear;                  // Очищаем список вариантов автозаверешения.
    for i := 0 to FDictionary.Count -1 do   // Перебираем строки из списка FDictionary.
      // Если FDictionary.Strings[i] подходит, добавляем его в список автозавершения.
      if CompareString(FindString, FDictionary.Strings[i]) then
          FDropDown.Items.Add(FDictionary.Strings[i]);
    // Сортируем найденные слова.
    FDropDown.Sort;
  end;
 
  procedure TAutoComplete.OutOfFocus(var M: TCMExit);
  begin
    inherited;
    // При уходе фокуса с TAutoComplete прячем список автозавершения.
    HideList(false);
  end;
 
  procedure TDropDown.CreateParams(var Params: TCreateParams);
  begin
    inherited CreateParams(Params);
    Params.ExStyle := WS_EX_TOOLWINDOW;
    Params.WndParent := GetDesktopWindow;
    Params.Style := WS_CHILD or WS_BORDER or WS_CLIPSIBLINGS or WS_OVERLAPPED or WS_VSCROLL;
  end;
 
  procedure TDropDown.OnLButtonUp(var M: TWMLButtonUp);
  begin
    inherited;
    // При выборе элемента из списка автозавершения помещаем его в поле ввода
    // TAutoComplete и закрываем окно автозавершения.
    if ItemIndex <> -1 then
      TAutoComplete(Owner).HideList(true);
  end;
 
  // Метод используется для подсвечивания в ListBox'е элемента, над которым
  // находится курсор мыши.
  procedure TDropDown.OnMouseOver(var M: TWMMouseMove);
  var p: TPoint;
      i: integer;
  begin
    inherited;
    // При движении мыши над выпадающем окном, подсвечиваем элемент, над которым
    // находится курсор мыши.
    GetCursorPos(p);
    p := ScreenToClient(p);
    i := ItemAtPos(p, true);
    if i <> -1 then
      begin
        ItemIndex := i;
        // Для того чтобы дальше обработывать событие OnChange у TAutoComplete,
        // необходимо обратно вернуть фокус на поле ввода.
        TAutoComplete(Owner).SetFocus;
      end;
  end;
 
  procedure TDropDown.OnActivateApp(var M: TMessage);
  begin
    inherited;
    // При переключении на другое приложение прячем окно автозавершения.
    if TAutoComplete(Owner).FDropped then
      TAutoComplete(Owner).HideList(false);
  end;
 
  // Процедура сортировки совпадающих слов
  // для показа наиболее подходящих слов первыми.
  procedure TDropDown.Sort;
  var Sorted: TStringList;
  begin
    // Создаём временный объект TListBox.
    Sorted := TStringList.Create;
    // Копируем содержимое элемента TDropDown.
    Sorted.Assign(TStrings(Items));
    // Сортируем список.
    Sorted.Sort;
    // Возвращаем результат.
    Items.Assign(Sorted);
    // Удаляем временный список.
    Sorted.Free;
  end;
 
  // Простое динамическое формирование пунктов контекстного меню.
  constructor TMenu.Create(AOwner: TComponent);
  begin
    inherited Create(AOwner);
    AddWord := TMenuItem.Create(AOwner);
    AddWord.Caption := 'Добавить';
    AddWord.OnClick := AddWordClick;
    Items.Add(AddWord);
    Items.Add(NewLine);
    Cut := TMenuItem.Create(AOwner);
    Cut.Caption := 'Вырезать';
    Cut.OnClick := CutClick;
    Items.Add(Cut);
    Copy := TMenuItem.Create(AOwner);
    Copy.Caption := 'Копировать';
    Copy.OnClick := CopyClick;
    Items.Add(Copy);
    Paste := TMenuItem.Create(AOwner);
    Paste.Caption := 'Вставить';
    Paste.OnClick := PasteClick;
    Items.Add(Paste);
    Delete := TMenuItem.Create(AOwner);
    Delete.Caption := 'Удалить';
    Delete.OnClick := DeleteClick;
    Items.Add(Delete);
    SelectAll := TMenuItem.Create(AOwner);
    SelectAll.Caption := 'Выделить всё';
    SelectAll.OnClick := SelectAllClick;
    Items.Add(SelectAll);
    // Событие на показ меню.
    OnPopup := OnPopupMenu;
  end;
 
  procedure TMenu.OnPopupMenu(Sender: TObject);
  begin
    // Если ничего не выделено в поле ввода, то выключаем пункт AddWord.
    // Проверяем длину выделенного текста.
    // SelLength может принимать и отрицательные значения.
    if TAutoComplete(Owner).SelLength <> 0 then
      AddWord.Enabled := true
    else
      AddWord.Enabled := false;
    // Если показан список автозавершения, то скрываем его.
    if TAutoComplete(Owner).FDropped then
      TAutoComplete(Owner).HideList(false);
  end;
 
  // Процедура при выборе пункта добавления строки.
  procedure TMenu.AddWordClick(Sender: TObject);
  begin
    TAutoComplete(Owner).AddWord;
  end;
 
  // Стандартные действия контексного меню поля ввода при выборе пунктов,
  // в комментариях не нуждаются.
  procedure TMenu.CutClick(Sender: TObject);
  begin
    TAutoComplete(Owner).CutToClipboard;
  end;
 
  procedure TMenu.CopyClick(Sender: TObject);
  begin
    TAutoComplete(Owner).CopyToClipboard;
  end;
 
  procedure TMenu.PasteClick(Sender: TObject);
  begin
    TAutoComplete(Owner).PasteFromClipboard;
  end;
 
  procedure TMenu.DeleteClick(Sender: TObject);
  begin
    TAutoComplete(Owner).ClearSelection;
  end;
 
  procedure TMenu.SelectAllClick(Sender: TObject);
  begin
    TAutoComplete(Owner).SelectAll;
  end;
 
end.
```
