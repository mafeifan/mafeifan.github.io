---
title: 看MDN的学到的总结
date: 2017-08-02 22:03:10
tags: javascript
---



通过 https://developer.mozilla.org 能学到不少web知识。现总结如下，不定期更新

### HTMLElement.dataset属性

html元素有一个全局属性。dataset。通过`data-yourCustom`的方式给DOM元素设置属性，然后通过
`dataset.yourCustom` 读取或设置属性值

看下面的例子中以`data-`开头的属性

<div id="user" data-id="1234567890" data-user="johndoe" data-date-of-birth>John Doe</div>


		``` javascript
		var el = document.querySelector('#user');

		// el.id == 'user'
		// el.dataset.id === '1234567890'
		// el.dataset.user === 'johndoe'
		// el.dataset.dateOfBirth === ''

		el.dataset.dateOfBirth = '1960-10-03'; // set the DOB.

		// 'someDataAttr' in el.dataset === false

		el.dataset.someDataAttr = 'mydata';
		// 'someDataAttr' in el.dataset === true
		```

<!--more-->