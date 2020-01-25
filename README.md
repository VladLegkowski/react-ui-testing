# Тестирование Реакт UI компонентов

## Что я делаю сейчас
В настоящее время я тестирую реализацию кода. Такие тесты ломаются каждый раз после рефакторинга, **особенно** в компонентах пользовательского интерфейса. В итоге я провожу кучу времени, копаясь в файлах `.test.js`, паралельно приследуюя магическую цифру в 80% для *Test Coverage*.
## Что я должен делать
При написании любого типа тестов, включая модульное тестирование, я должен меньше думать о коде который я тестирую, а больше о том, что делает данный код. Это означает писать тесты, имитирующие поведение пользователя. Даже на самом низком уровне.
<cut/>
## Пример
Представим стандартный юай компонент аккордеон. При нажатии он раскрывается или закрывается. Контент передается в компонента как `children`. 

![image](https://habrastorage.org/webt/zp/gp/e2/zpgpe2iyiknldvwwvmhcm_2yzxi.png)

Функционал нашего, тестового, компонента выглядит следующим образом:

 - Первые 3 аккордеона развернуты, все оставшиеся закрыты.
 - При нажатии на аккордеон срабатывает модуль `publishAccordionAnalytics`, который трекерит аналитику. Данный модуль мы импортируем из пакета`@myProject/analyticsHelpers`
 - Если пользователь нажимает на скрытый аккордеон в нижней части приложения, и после раскрытия содержание аккордеона не находится в поле зрения пользователя, срабатывает анимация и контент компонента выезжает в видимую часть экрана.

```javascript
...
import { publishAccordeonAnalytics } from '@myProject/analyticsHelpers';

class Accordion extends Component {

      positionReference: RefObjectType;

      constructor(props) {

        super(props);

        this.positionReference = React.createRef();

      }

      scrollToRef = (ref) => {

        const wrapper = document.querySelector('.app');

        return isInViewPort(ref, wrapper)

          ? null

          : setTimeout(() => {

            return ref.current

              && wrapper

              && wrapper.scrollTo(wrapper.scrollTop, wrapper.scrollTop + ref.current.offsetTop)

          }, 300);

      };

      render() {

        const { headerName, children, index } = this.props;

        const { scrollToRef, positionReference } = this;

        const defaultOpenAccordions = index >= 3;

        return (

          <Accordion

            defaultOpen={!defaultOpenAccordions}

            headerName={headerName}

            onChange={(open) => {

              publishAccordionAnalytics(open, headerName);

              if (defaultOpenAccordions) {

                scrollToRef(positionReference);

              }

            }}

            id={headerName}

          >

            {children}

            {defaultOpenAccordions && <div data-testId="referenced-div" ref={positionReference} />}

          </Accordion>

        );

      }

    }
```
Как я напишу тест сейчас:
```javascript
jest.mock('@myProject/analyticsHelpers');

    describe('Аккордеон', () => {

      test('воспроизводится верно', () => {

        const jsx = (

            <Accordion headerName="Имя аккордеона">

              <div>Test</div>

            </Accordion>

        );

        const tree = renderer

          .create(jsx)

          .toJSON();

        expect(tree).toMatchSnapshot();

      });

      test('аналитика вызвана', () => {

        const wrapper = mount(

            <Accordion headerName="Имя аккордеона" index={1}>

              <div>Test</div>

            </Accordion>

        );

        wrapper.find('Header').simulate('click');

        expect(publishAccordeonAnalytics).toHaveBeenCalledTimes(1);

      });

      test('функция scrollToRef вызвана', () => {

        const wrapper = shallow(

          <Accordion headerName="Имя аккордеона" index={7}>

            <div>Test</div>

          </Accordion>);

        const component = wrapper.instance();

        component.scrollToRef = jest.fn();

        wrapper.find('Header').simulate('click');

        expect(publishAccordeonAnalytics).toHaveBeenCalledTimes(1);

        expect(component.scrollToRef).toHaveBeenCalled();

      })

    });
```
Я тестирую структуру компонента при помощи снапшота, а также вызовы функции при клике. 

Данный аккордеон полностью отвечает ожидаемому функционалу и тесты это подтверждают. Но теперь я сделаю рефакторинг, заменив `React.Component` на `functional component`, и вынесу метод компонента `scrollToRef` в отдельную функцию.

```javascript
function Accordion ({ marketName, children, index }) {

      const positionReference = React.createRef();

      const defaultOpenAccordions = index >= config.defaultOpenAccordions;

      return (

        <Accordion

          defaultOpen={!defaultOpenAccordions}

          headerName={headerName}

          onChange={(open) => {

            publishAccordeonAnalytics(open, marketName);

            if (defaultOpenAccordions) {

              scrollToRef(positionReference);

            }

          }}

          id={marketName}

        >

          {children}

          {defaultOpenAccordions && <div data-testId="referenced-div" ref={positionReference} />}

        </Accordion>

      );

    }
```
Мои тесты посыпались... тест`'функция scrollToRef вызвана'` терпит неудачу, т.к. функция `scrollToRef` больше не является частным методом компонента. То же самое случилось бы с тестом `'аналитика вызвана'`, но он является импортом из модуля, так сейчас он pass.

Чтобы написать хороший тест, мне надо понять, как юзер использует мой компонент. Юзер:

 1. Нашел аккордеон, который содержит нужную ему информацию
 2. Нажал на название аккордеона, чтобы его открыть
 3. Прочитал ифну
 4. Закрыл аккордеон

![](https://habrastorage.org/webt/6y/u1/vi/6yu1vifp7wdz0jyswlep51ryk3y.gif)

Я понял, что его вообще не волнует, как называется мой компонент, на что именно он нажимает и так далее. Следуя этому принципу, я должен был написать что-то вроде этого:

```javascript
import '@testing-library/jest-dom/extend-expect';
import { render, fireEvent, screen } from '@testing-library/react';
jest.mock('@myProject/analyticsHelpers');

    describe('Аккордеон', () => {

      test('полная функциональность компонента', () => {

        const child = <div>Ребенок</div>;

       // аккордеон № 10 в списке

        const { getByText } = render(

          <Accordion headerName="аккордеон номер 10" index={9}>

            {child}

          </Accordion>

        );

        // открываю аккордеон

        fireEvent.click(getByText(/аккордеон номер 10/i));

        expect(publishAccordeonAnalytics).toHaveBeenCalled();

        expect(screen.queryByText('Ребенок')).toBeInTheDocument();

        expect(screen.getByTestId('referenced-div')).toBeInTheDocument();

        // закрываю аккордеон

        fireEvent.click(getByText(/аккордеон номер 10/i));

        expect(publishAccordeonAnalytics).toHaveBeenCalled();

      });

    });
```
Теперь я воспроизвел поведение пользователя. Нашел 10й аккордеон, нажал на него, прочитал контент, закрыл его - всё!

Этот тест визуально намного чище, выдержит рефакторинг и имитирует взаимодействие пользователей. И мне была их намного проще написать.

Используя данный паттерн мы сможем избежать использования энзаймовских нативных `instance()`, `state()`, `find('ComponentName')` и иных функций, тестирующих реализацию кода.

### Для нового теста я использовал

* [Jest](http://facebook.github.io/jest/)
* [React Testing Library](https://testing-library.com/)
