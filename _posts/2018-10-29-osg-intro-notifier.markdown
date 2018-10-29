---
layout: post
title:  "Введение в OpenSceneGraph: Трассировка, уведомления, логирование"
date:   2018-10-29 21:50:00 +0300
categories: jekyll update
---

OpenSceneGraph имеет механизм уведомлений, позволяющий выводить отладочные сообщения в процессе выполнения рендеринга, а так же инициированные разработчиком. Это серьезное подстпорье при трассировке и отладке программы. Система уведомлений OSG поддерживает вывод диагностической информации (ошибки, предупреждения, уведомления) на уровне ядра движка и плагинов к нему. Разработчик может вывести диагностическое сообщение в процессе работы программы, воспользовавшись функцией osg::notify().

Данная функция работает как стандартный поток вывода стандартной бибиотеки C++ через перегрузку оператора <<. В качестве аргумента она принимает уровень сообщения: ALWAYS, FATAL, WARN, NOTICE, INFO, DEBUG_INFO и DEBUG_FP. Например

```cpp
osg::notify(osg::WARN) << "Some warning message" << std::endl;
```
выводит предупреждение с определенным пользователем текстом.

## Перенаправление уведомлений
