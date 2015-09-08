---
layout: post
title: "enron topic modelling"
date: 2015-09-04 14:34:37 +0300
comments: true
categories: 
---

$$

\usepackage{amsmath}
\usepackage{mathtools}
\DeclarePairedDelimiter{\vert}{\lvert}{\rvert}

$$

Есть такая задача --- *тематическое моделирование* --- заключающаяся в построении *тематической модели* заданной коллекции документов, представленных в текстовом виде (текстовый корпус). В свою очередь, *тематическая модель* --- модель, определяющая принадлежность документов различным темам. Например, рассматривая в качестве коллекции документов новости, одной из тем может оказаться спорт, другой --- шоу-бизнес, третьей --- ~~санкции в ответ на санкции~~ внешняя политика и т.д.

Если пока что не очень понятно, что же такое тематическое моделирование, то ничего страшного: далее в этой статье мы рассмотрим применения нескольких алгоритмов тематического моделирования к текстовому корпусу [Enron Email Dataset](https://www.cs.cmu.edu/~./enron/). Начнем же мы с описания этого корпуса.

## Enron Email Dataset
Начнем с предыстории. [Enron](https://en.wikipedia.org/wiki/Enron) --- американция энергетическая корпорация, являвшаяся одной из крупнейших в мире по транспортировке электроэнергии, природного газа и бумажной промышленности вплоть до ее банкротства в декабре 2001 года. Причина этого банкротства, ставшего одним из купнейших в истории США, заключалась в корпоративном мошенничестве и коррупции направленных на фальсификацию финансовой отчетности. В ходе разбирательства Федеральной Комиссией по Регулированию Энергетики были обнародаванны электронные письма примерно 150 сотрудников Enron, преимущественно высшего руководства. В настоящий момент, уже предобработанная коллекция насчитывающая порядка полумиллиона писем находится в [открытом доступе](https://www.cs.cmu.edu/~./enron/).
Таким образом, Enron Email Dataset является одним из крупнейших корпусов писем в открытом доступе, зачастую используемым исследователями для аппробации новейших алгоритмов, в том числе и методов тематического моделирования.
В этом наборе данных сообщения расположены в различных директориях для каждого из пользователей (так, одно и то же сообщение хранится в папке "исходящие" для отправителя и папке "входящие" для получателя). Рассмотрим пример сообщения

```python Исходный формат
Message-ID: <15033497.1075840963660.JavaMail.evans@thyme>
 Date: Mon, 7 Jan 2002 12:58:33 -0800 (PST)
 From: louise.kitchen@enron.com
 To: laura.luce@enron.com
 Subject: Peoples
 Cc: jim.fallon@enron.com
 Mime-Version: 1.0
 Content-Type: text/plain; charset=us-ascii
 Content-Transfer-Encoding: 7bit
 Bcc: jim.fallon@enron.com
 X-From: Kitchen, Louise </O=ENRON/OU=NA/CN=RECIPIENTS/CN=LKITCHEN>
 X-To: Luce, Laura </O=ENRON/OU=NA/CN=RECIPIENTS/CN=Lluce>
 X-cc: Miller, Don (Asset Mktg) </O=ENRON/OU=NA/CN=RECIPIENTS/CN=Dmille2>, Fallon, Jim </O=ENRON/OU=NA/CN=RECIPIENTS/CN=Jfallon>
 X-bcc: 
 X-Folder: \ExMerge - Kitchen, Louise\Sent Items
 X-Origin: KITCHEN-L
 X-FileName: louise kitchen 2-7-02.pst
 
 In order to get Peoples up and running as soon as possible - we will need (as Netco) to enter into a service agreement with the Estate - can you set this up with Don or Jim.
 
 Louise Kitchen
 Chief Operating Officer
 Enron Americas
 Tel:  713 853 3488
 Fax: 713 646 2308
```

 Как видно, в сообщении довольно много служебной информации. Как известно, большую часть своего времени data scientist тратит на очистку и преобразование данных. Дабы избежать этот не самый увлекательный процесс вместо этого "сырого" корпуса мы возьмем уже [обработанный и представленный в формате "mongodump"](http://mongodb-enron-email.s3-website-us-east-1.amazonaws.com/). Здесь каждое письмо представленно в виде json, например:

```json JSON-формат
 {u'_id': ObjectId('4f16fd69d1e2d3237103fcbe'),
 u'body': u'In order to get Peoples up and running as soon as possible - we will need (as Netco) to enter into a service agreement with the Estate - can you set this up with Don or Jim.\n\nLouise Kitchen\nChief Operating Officer\nEnron Americas\nTel:  713 853 3488\nFax: 713 646 2308',
 u'filename': u'349.',
 u'headers': {u'Bcc': u'jim.fallon@enron.com',
  u'Cc': u'jim.fallon@enron.com',
  u'Content-Transfer-Encoding': u'7bit',
  u'Content-Type': u'text/plain; charset=us-ascii',
  u'Date': u'Mon, 7 Jan 2002 12:58:33 -0800 (PST)',
  u'From': u'louise.kitchen@enron.com',
  u'Message-ID': u'<15033497.1075840963660.JavaMail.evans@thyme>',
  u'Mime-Version': u'1.0',
  u'Subject': u'Peoples',
  u'To': u'laura.luce@enron.com',
  u'X-FileName': u'louise kitchen 2-7-02.pst',
  u'X-Folder': u'\\ExMerge - Kitchen, Louise\\Sent Items',
  u'X-From': u'Kitchen, Louise </O=ENRON/OU=NA/CN=RECIPIENTS/CN=LKITCHEN>',
  u'X-Origin': u'KITCHEN-L',
  u'X-To': u'Luce, Laura </O=ENRON/OU=NA/CN=RECIPIENTS/CN=Lluce>',
  u'X-bcc': u'',
  u'X-cc': u'Miller, Don (Asset Mktg) </O=ENRON/OU=NA/CN=RECIPIENTS/CN=Dmille2>, Fallon, Jim </O=ENRON/OU=NA/CN=RECIPIENTS/CN=Jfallon>'},
 u'mailbox': u'kitchen-l',
 u'subFolder': u'sent_items'}
```


Как видно, здесь все содержимое письма полностью разобранно на такие составляющие, как: почтовая директория, отправитель и получатели, непосредственно текст письма и прочее. В дальнейшем это значительно упростит нам работу. 


Вернемся к задаче тематического моделирования --- построения модели документов, которая определяет

1. к каким темам относится каждый из документов,
2. какие слова образуют каждую из тем.

Вопрос: как же построить алгоритм, который автоматически отвечает на эти два вопроса для заданного набора текстовых документов? Ответ: нужно призвать на помощь математику. К сожалению, математике не знакомые понятия "текст", "документ", "слово" и "тема". Зато, ей хорошо понятен язык чисел и векторов. И именно на язык векторов нам и предстоить перейти для решения нашей задачи. 

## Векторная модель
Векторная модель --- модель представления текстовых документов в виде числовых векторов, где каждое измерение вектора соответствует какому-либо слову. Строится эта модель следующим образом.

Рассмотрим коллекцию документов $$\mathbf{D} = \{\mathbf{d}\}$$ (корпус) и словарь слов $$\mathbf{W} = \{\mathbf{w}\}$$. Каждой паре $$(\mathbf{d},\mathbf{w})$$ можно сопоставить число $$n_{\mathbf{d},\mathbf{w}}$$ --- количество появлений слова $$\mathbf{w}$$ в документе $$\mathbf{d}$$. 
Теперь, пронумеруем документы числами от 1 до $$\lvert\mathbf{D}\rvert$$ и слова числами от 1 до $$\lvert\mathbf{W}\rvert$$ так, что документу $$\mathbf{d} \in \mathbf{D}$$ соответствует номер $$d\in\{1,\ldots,\lvert\mathbf{D}\rvert\}$$, а слову $$\mathbf{w} \in \mathbf{W}$$ --- номер $$w\in\{1,\ldots, \lvert\mathbf{W}\rvert\}$$. 
Таким образом, $$n_{w,d}=n_{\mathbf{w},\mathbf{d}}$$ можно сформировать матрицу $$X = (n_{d,w})_{d,w}$$ --- матрица частот слов в документах, строки которой соответствуют документам, а столбцы --- словам.


Подобная векторная модель опирается на следующие предположения

- Порядок документов в коллекции не имеет значения
- Порядок слов в документе не имеет значения, документ — мешок слов
- Коллекцию документов можно представить как выборку пар документ-слово $$(d, w)$$, $$d \in D$$, $$w \in \mathit{W}_d$$.
- Cлово в разных формах — это одно и то же слово
- Слова, встречающиеся часто в большинстве документов, не важны для определения тематики

Зададим матрицу $$\mathbf{W}$$ следующим образом

$$
    (\mathbf{W})_{d,w} =  n_{d,w},
$$

