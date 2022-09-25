<!--
 * @Description: 
-->
#### CSS Modules
css 是前端领域中进化最慢的一块。由于es6的快速普及以及Babel与webpack等工具的迅速发展，相较于JavaScript，CSS被远远甩在后面，逐渐成为各类大型项目工程化的痛点，也成了前端走向彻底模块化前必须要解决的一个难题。

CSS 模块化的解决方案有很多，但主要有两类。
- inline Style。这种方案彻底抛弃CSS，使用JavaScript或JSON来写样式，能给CSS提供JavaScript同样强大的模块化能力。但缺点同样明显，Inline Style几乎不能利用CSS本身的特性，比如联级、媒体查询等，:hover和:active等伪类处理起来比较复杂。另外，这种方案需要依赖框架实现，其中与React相关的Radium、jsxstyle和react-style。

-  CSS Modules。 依旧使用CSS，但使用JavaScript来管理样式依赖。CSS Modules能最大化地结合现有CSS生态和JavaScript模块化能力，其API非常简洁，学习成本几乎为零。发布时依旧编译出单独的JavaScript和CSS文件。现在， webpack css-loader内置CSS Modules 功能。

1. CSS 模块化遇到了哪些问题？
  css 模块化重要的是解决好以下两个问题：css 样式的导入与导出。灵活按需导入以便复用码，导出时要能够隐藏内部作用域，以免造成全局污染。Sass、Less、PostCSS等试图解决CSS编程能力弱的问题，但这并没有解决模块化这个问题。React开发中遇到的一系列css相关问题，结合实际开发的问题有以下几点。
    - 全局污染：CSS 使用全局选择器机制来设置样式，优点是方便重写样式。缺点是所有的样式都是全局生效，样式可能被错误覆盖，因此产生了非常丑陋的！important,甚至inline !important和复杂的选择器权重计数表，提高犯错概率和使用成本。Web Components标准中的Shadow DOM能彻底解决这个问题，但它把样式彻底局部化，造成外部无法重写样式，损失了灵活性。
    - 命名混乱：由于全局污染的问题，多人协同开发时为了避免样式冲突，选择器越来越复杂，容易形成不同的命名风格，很难统一。样式变多后，命名将更加混乱。
    - 依赖管理不彻底：组件应该相互独立，引入一个组件时，应该只引入它所需要的css样式。现在的做法是除了要引入JavasScript,还要再引入它的CSS,而且Sass/Less很难实现对每个组件都编译出单独的CSS，引入所有模块的CSS又造成浪费。JavaScript的模块化已经非常成熟，如果能让JavaScript来管理CSS依赖是很好的解决办法，而webpack的css-loader提供了这种能力。
    - 无法共享变量： 复杂组件要使用JavaScript和CSS来共同处理样式，就会造成有些变量在JavaScript和CSS中冗余，而预编译语言不能提供跨JavaScript和CSS共享变量的这种能力。
    - 代码压缩不彻底: 由于移动端网络的不确定性，现代工程项目对CSS压缩的要求已经到了变态的程度。很多压缩工具为了节省一个字节，会把16px转换成1pc,但是这对非常长的类名却无能为力。

    上述问题只凭CSS自身是无法解决的，如果通过JavaScript来管理CSS，就很好解决。因此，给出的解决方案是完全的CSS in JS，但这相当于完全抛弃CSS，在JavaScript中以hash映射来写CSS，但这种做法未免有些激进，直到出现了CSS Modules。

2. CSS Modules 模块化方案
  CSS Modules 内部通过ICSS来解决样式导入和导出这两个问题，分别对应:import和:export两个新增的伪类：
  ```JavaScript
    :import('path/to/dep.css') {
      localAlias: keyFromDep;
      /* ... */
    }
    :export {
      exportedKey: exportedValue;
      /* ... */
    }
  ```
  但直接使用这两个关键字编程太繁琐，项目中很少直接使用他们，我们需要的是用JavaScript来管理CSS的能力。结合webpack的css-loader，就可以在css中定义样式，在JavaScript文件中导入。

  - 启用CSS Modules
  启用CSS Modules的代码如下：
  ```JavaScript
    // webpack.config.js
    css?modules&localIdentName=[name]__[local]-[hash:base64:5]
  ```
  加上modules即为启用，其中localIdentName是设置生成样式的命名规则。

  下面我们直接看看怎么引用css, webpack又是怎么转化class名的：

  ```JavaScript
    /* components/Button.css */
    .normal {
      /* normal相关的所有样式 */
    }  
    .disabled {
      /* 
        disabled 相关的所有样式
      */
    }
  ```
  将以上css保存好，然后用import的方式在javascript文件中引用：
  ```JavaScript
      /* components/Button.js */
      import styles from './Button.css'

      console.log(styles);
      buttonElem.outerHTML = `<button class=${styles.normal}>Submit</button>`
  ```
  我们看到，最终生成的HTML是这样的：
  ```html
    <button class="button--normal-abc5436">Processing ...</button>
  ```
  注意到button--normal-abc5436 是CSS Modules按照 localIdentName自动生成的class名称，其中abc5326是按照给定算法生成的序列码。经过这样混淆处理后，class的名称基本就是唯一的，大大降低了项目中样式覆盖的几率。同样在生成环境下修改规则，生成更短的class名，可以提高css的压缩率。
  css modules 对css中的class名都做了处理，使用对象来保存原class和混淆后class的对应关系。通过这些简单的处理，css modules实现了以下几点：
  - 所有样式都是局部化的，解决了命名冲突和全局污染问题；
  - class名的生成规则配置灵活，可以以此来压缩class名;
  - 只需要引用组件的javascript，就够搞定组件所有的javascript和css
  - 依然是css,学习成本几乎为零
<br />

- __样式默认局部__
  使用了css modules后，就相当于给每个class名外加了:local，以此来实现样式的局部化。如果我们想切换到全局模式，可以使用:global包裹。示例代码如下：
  ```JavaScript
      .normal {
        color: green;
      }
      /* 以上与下面等价 */
      :local(.normal) {
        color: red;
      }

      /* 定义全局样式 */
      :global(.btn) {
        color: blue;
      }

      /* 定义多个全局样式 */
      :global {
        .link {
          color: green;
        }
        .box {
          color: yellow;
        }
      }
  ```
  - 使用composes来组合样式
  对于样式复用，css modules只提供了唯一的方式来处理--composes组合。示例：
  ```JavaScript
      /* components/Button.css */
      .base { /* 所有通用的样式 */}
      .normal {
        composes: base;
        /* normal其他样式 */
      }

      .disabled {
        composes: base;
        /* disabled 其他样式 */
      }

      import styles from './Button.css';

      buttonElem.outerHTML = `<button class=${styles.normal}>Submit</button>`
  ```
  生成的HTML变为：
  ```html
    <button class="button--base-abc53 button--normal-abc53">Processing ...</button>
  ```
3. CSS Modules使用技巧
  CSS Modules 是对现有的css做减法。为了追求简单可控，作者建议遵循如下原则：
  - 不适用选着器，只使用class名来定义样式；
  - 不层叠多个class, 只使用一个class把所有样式定义好；
  - 所有样式通过composes组合来实现复用；
  - 不嵌套

  例举一些常见问题：
  1. 如果对一个元素使用多个class呢？
  样式照样生效
  2. 在一个style文件中使用同名的class呢？
  这些同名class编译后虽然可能是随机码，但仍然是同名的。
  3. 在style文件中使用了id选择器、伪类和标签选着器等呢？
  所有这些选择器将不被转换，原封不动地出现在编译后的css中。也就是说，css modules只会转换class名相关的样式。
  
  - 外部如何覆盖局部样式
  当生成混淆的class名后，可以解决命名冲突，但因为无法预知最终的class名，不能通过一般选择器覆盖。我们可以给组件关键节点加上data-role属性，然后通过属性选择器来覆盖样式：
  ```jsx
    // dialog.js
    return (
      <div className={styles.root} data-role="dialog-root">
        <a 
          className={styles.disabledConfirm}
          data-role="dialog-confirm-btn"
        >
          Confirm
        </a>
        ...  
      </div>
    )

    // dialog.css
    [data-role = 'dialog-root'] {
      // override style
    }
  ```
  因为css modules只会转变类选择器，所以这里的属性选择器不需要添加:global。

  - css modules结合react实践
  在className处直接使用css中的class名即可：
  ```jsx
      /* dialog.css */
      .root {}
      .confirm {}
      .disabledConfirm {}

      /* 
        dialog.js
      */
     import React, { Component } from 'react'
     import classnames from 'classnames'
     import styles from './dialog.css'

     class Dialog extends Component {
        render() {
          const { disabled } = this.state
          const cx = classNames({
            confirm: !disabled
            disabledConfirm: disabled
          })

          return (
            <div className={styles.root}>
              <a className={styles[cx]}>Confirm</a>
            </div>
          )
        }
     }
  ```
  如果不想频繁地输入style.\**, 可以使用react-css-modules库。它通过高阶组件的形式避免重复输入styles.**。重写上面的例子：
  ```jsx
    import React, { Component } from 'react'
    import classNames from 'classnames'
    import CSSModules from 'react-css-modules'
    import styles from './dialog.css'

    class Dialog extends Component {
      render() {
        const { disabled } = this.state
        const cx = classNames({
          confirm: !disabled
          disabledConfirm: disabled
        })

        return (
          <div styleName="root">
            <a styleName={cx}></a>
          </div>
        )
      }
    }

    export default CSSModules(Dialog, styles)
  ```
  此外，对比原始的CSS Modules，有以下几个优点：
  - 我们不用再关注是否使用驼峰来命名class名；
  - 我们不用每一次使用css modules 的时候都关联style对象
  - 使用css modules,容易使用:global去解决特殊情况，使用react-css-modules可以写成<div className="global-css" styleName="local-module"></div>,这种形式轻松对应全局和局部：
  - 当styleName关联了一个undefined CSS Modules时，我们会得到一个警告：
  - 我们可以强迫使用单一的CSS Modules。