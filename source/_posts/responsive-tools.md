---
title: Техники для комфортного написания респонсивного кода (неоконченно)
date: 2016-07-13 20:42:01
tags: tools, mixins, responsive
---


## Проблема
TODO


## Обзор существующих методов
TODO


## Что предлагаю
TODO


## Как это работает
TODO

### Фаза 1: готовим почву

#### Шаг 1.1
Давайте для начала абстрагируемся от пикселей. 
Создадим глобальный объект переменных с которым будет работать наш код

{% codeblock lang:stylus %}
$settings = {
  gap: 5px
  breakpoints: {
    lg: 1200px
    md: 960px
    sm: 768px
    xs: 480px
    xxs:320px
  }
}
{% endcodeblock %}

#### Шаг 1.2
Теперь добавим блочные миксины для более быстрых медиа запросов

{% codeblock lang:stylus %}
max(bp)
  condition = 'only screen and (max-width: %s)' % ($settings.breakpoints[s('%s', bp)] or bp)
  @media condition
    {block}
 
min(bp)
  condition = 'only screen and (min-width: %s)' % ($settings.breakpoints[s('%s', bp)] or bp)
  @media condition
    {block}  
{% endcodeblock %}

##### Обратите внимание, что эти миксины умеют принимать в качестве аргумента ключи объекта $settings.breakpoints. Это нам впоследствии очень пригодится.


### Фаза 2: пишем миксины

#### Шаг 2.1
Давайте зададим локальные переменные, которые будет использовать наш миксин. 
Остановимся пока на флексбоксах.


{% codeblock lang:stylus %}
$setup = {
  align: {
    x: {
      '1': flex-start
      '2': center
      '3': flex-end
      '13': space-between
    }
    y: {
      '1': flex-start
      '2': center
      '3': flex-end
      '13': stretch
      '21': baseline
    }
  }
  dir: {
    x: row
    y: column
    rx: row-reverse
    ry: column-reverse
  }
}
{% endcodeblock %}

##### Нумерические ключи необходимо задавать в строковом формате, иначе произойдет ошибка при компиляции
Основная идея этого блока - абстракция и упрощение. Нужно несколько месяцев поработать с флексбоксами чтобы, наконец, выучить все значения *align-items* и *justify-content*, да и перестать путать их самих :)
Лично я нашел выход в специальной нотации на основе 3x3 матрицы. Но о ней ниже.


### Шаг 2.2
Пишем миксины, которые будут преобразовывать конфиг в css свойства

{% codeblock lang:stylus %}
set-align()
  y = arguments[0]
  x = arguments[1] || arguments[0]
  justify-content: $setup.align.x['' + x]
  align-items: $setup.align.y['' + y]

set-dir(d)
  flex-direction: $setup.dir[d]

set-fill(f)
  if f
    flex 1 0 0%
    max-width 100% //fix FF and IE content overflowing issue
    min-width: 0  //fix ellipsis bug inside flex containers at FF & Chorme 48+
  else
    flex 0 1 auto

set-grid()
  for col, i in arguments
    &>*:nth-child({i+1}n)
      -set-fill true
      flex-grow col
{% endcodeblock %}

Кратко пройдусь по коду:
- set-align контролирует выравнивание по *main axis* (x) и *cross-axis* (y)
- set-dir устанавливает *main axis* (следует заметить, что в случае значений *column* или *column-reverse* set-align будет работать "шиворот-навыворот" :)
- set-fill принимет булево значение и контролирует способность элемента заполнять все возможное пространство
- set-grid сетка вдохновленная CSS3 Grid Layout module. 


### Шаг 2.3
Вот мы добрались и до самого конфигурационного миксина. Он до смешного простой:

{% codeblock lang:stylus %}
  setup(config = {})
    display flex
    for prop, value in config
      set-{prop}: value
{% endcodeblock %}

Для каждого свойства объекта, переданного вами в качестве аргумента, вызывается соответствующий миксин. 

Например, простое выравнивание дочерних элементов по высоте и ширине родителя. Расположение элементов по оси X (друг за другом).

{% codeblock lang:stylus %}
.align-all-content-at-center
  setup({
    dir: x
    align: 2 2
  })
{% endcodeblock %}

Компилится в: (для наглядности вендорные префиксы опущены)

{% codeblock lang:css %}
.align-all-content-at-center {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
}
{% endcodeblock %}

Рассмотрим еще один распространенный случай: две колонки выровнянные по высоте из которых левая (первая) в два раза больше правой:

{% codeblock lang:stylus %}
.layout
  setup({
    dir: x
    align: 1 13
    grid: 2 1
  })
{% endcodeblock %}

Результат:

{% codeblock lang:css %}
.layout {
  display: flex;
  flex-direction: row;
  justify-content: flex-start;
  align-items: stretch;
}
.layout>*:nth-child(1n) {
  flex-grow: 2;
}
.layout>*:nth-child(2n) {
  flex-grow: 1;
}
{% endcodeblock %}



### Шаг 2.3 Из Агмона в Греймона
Давайте подытожим: у нас уже есть работающий миксин, который позволяет с легкостью оперировать раскладкой внутри элемента. Неплохо. Но давайте создадим его улучшенную версию, которая позволит работать с респонсивными моделями. 

{% codeblock lang:stylus %}
rSetup(configs = {})
  for name, config in configs
    if $bp[name] and config
        +min($bp[name])
          setup(config)
{% endcodeblock %}

Пример использования красноречив: 

{% codeblock lang:stylus %}
.responsive-layout
  rSetup({
    xxs: {
      align: 2 2
      dir: y
    }
    sm: {
      align: 2 13
      dir: x
    }
  })
{% endcodeblock %}

Пример использования красноречив: 

{% codeblock lang:css %}
@media only screen and (min-width: 320px) {
  .responsive-layout {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }
}
@media only screen and (min-width: 768px) {
  .responsive-layout {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    align-items: center;
  }
}
{% endcodeblock %}

Мы создали раскладку, которая выравнивает по ширине/высоте и располагает друг под другом на всех экранах до *$settings.breakpoints.sm*, т.е. до *768px*. В то же время если ширина окна больше этого значения - элементы располагаются друг за другом и выравниваются по центру высоты и по краям ширины блока-родителя (да простят меня боги за этот ужасный оборот :)
