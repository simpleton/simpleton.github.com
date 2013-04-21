---
author: sunsj1231
comments: true
date: 2011-08-11 09:12:21
layout: post
slug: qmlqml_declare_type-macro-trap
title: '[QML]QML_DECLARE_TYPE macro trap'
wordpress_id: 44059
---

QML_DECLARE_TYPE Equivalent to Q_DECLARE_METATYPE(TYPE) and Q_DECLARE_METATYPE(QDeclarativeListProperty<TYPE>)

Q_DECLARE_METATYPE ( Type ) This macro makes the type Type known to QMetaType as long as ****it provides **a public default constructor, a public copy constructor and a public destructor**. It is needed to use the type Type as a custom type in QVariant.

Ideally, this macro should be placed below the declaration of the class or struct. If that is not possible, it can be put in a private header file which has to be included every time that type is used in a [QVariant](qvariant.html).

Adding a Q_DECLARE_METATYPE() makes the type known to all template based functions, including[QVariant](qvariant.html). Note that if you intend to use the type in _queued_ signal and slot connections or in [QObject](qobject.html)'s property system, you also have to call [qRegisterMetaType](qmetatype.html#qRegisterMetaType)() since the names are resolved at runtime



BUT it doesn`t work , i must call qmlRegisterType to register the target type to the MetaObject system. so confused~

wasting the whole night!!!
