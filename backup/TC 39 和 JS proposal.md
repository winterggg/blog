# TC 39 和 JS proposal

TC39（Technical Committee 39）是负责ECMAScript标准化的组织，它是ECMA International的一部分，后者是一个致力于信息和通信系统标准化的国际私有组织. 

TC39 是一个由 JavaScript 开发者、实现者、学者等组成的团体，与 JavaScript 社区合作维护和发展 JavaScript 的标准，ECMAScript是JavaScript的官方标准. 

总的来说，JavaScript 的标准库相对较小，但 TC39 的一个趋势是使 JS 成为一种“内置电池（batteries-included）”的语言，提供高质量的内置功能集. 例如，[Temporal](https://github.com/tc39/proposal-temporal) 正在取代 moment.js，一些小功能，如 `Array.prototype.flat` 和 `Object.groupBy` 正在取代许多 lodash 的使用场景. 



[JavaScript提案（JS proposal）](https://www.proposals.es/) 是指任何希望被加入ECMAScript标准的新功能、语法或者API的正式建议. 提案通过一系列的阶段，从一个初步想法发展成为标准的一部分. 

在讨论TC39和JavaScript提案时，有几个常见的术语：

1. **ECMAScript (ES)**: JavaScript的官方名称和标准，由TC39维护. 
2. **Proposal**: 一个提议，它旨在引入新的功能或对现有功能进行更改. 
3. **Champion**: 一个或一组TC39委员会成员，他们将某个提案视为有价值，并愿意承担起推动提案前进的责任. 他们会在TC39会议上代表提案，回答问题，处理反馈，并且确保提案在整个过程中不断进展. 通常，一个提案至少需要一个champion，但有时一个提案可能有一个小组的champion，尤其是对于复杂或者重要的更改. 
4. **Strawperson (Stage 0)**: 提案的最初阶段，用于讨论和收集对新特性的反馈. 任何想法都可以被当作Stage 0的提案，不需要形式化的规范. 
5. **Proposal (Stage 1)**: 提案已经正式提交并获得一位或多位champion的支持，提供了初步的规范草案和问题讨论，包括一个高级别的语言描述、示例、API、语义和算法等. 
6. **Draft (Stage 2)**: 提案有了初步的规范，并且不太可能有大的变化. 开发者和其他利益相关者开始关注提案的细节. 
7. **Candidate (Stage 3)**: 提案规范文本基本完成，并且已经有了多个实验性实现. 在这个阶段，提案需要实际的反馈来确定是否准备好成为标准的一部分. 
8. **Finished (Stage 4)**: 提案已经完成，在两个独立的实现中被证明是可行的，并且至少有一个通过了所有正式的ECMAScript兼容性测试套件（Test262）. 一旦提案达到这个阶段，就会被加入下一个版本的ECMAScript标准. 
9. **Consensus**: TC39的决策过程基于共识，这意味着提案必须取得委员会成员的普遍同意才能前进. 
10. **Polyfill/Shim**: 一个用来在老版本的JavaScript引擎中模拟新特性的代码片段，使得开发者可以提前使用新特性，而不用等到所有用户的浏览器都支持这些特性. 
11. **Transpiler**: 一个转换编译器，比如Babel，它能够将使用新ECMAScript特性的代码转换成向后兼容的JavaScript版本，以便可以在当前的JavaScript环境中运行. 
12. **Test262**: ECMA International维护的 ECMAScript [官方测试套件](https://github.com/tc39/test262)，用于验证符合性和实现的正确性.