# Rust API 指南

这是一套关于如何设计和呈现Rust编程语言API的建议。这些建议主要由Rust库团队撰写，基于构建Rust标准库和其他生态系统中的crate的经验。

这些只是指导原则，其中有些比其他的更为严格。在某些情况下，它们可能较为模糊且仍在发展中。Rust crate的作者应将其视为开发惯用且可互操作的Rust库时的重要考量，根据需要使用。这些指南不应被视为crate作者必须遵循的强制性规定，但他们可能会发现，遵循这些指南的crate比不遵循的更好地与现有的crate生态系统整合。

本书分为两部分：简明的[检查清单]，包含所有单独的指南，适合在crate审查时快速浏览；以及专题章节，详细解释各项指南。

如果你有兴趣为API指南做出贡献，请查看[contributing.md]并加入我们的[Gitter频道]。

[检查清单]: checklist.html
[contributing.md]: https://github.com/rust-lang/api-guidelines/blob/master/CONTRIBUTING.md
[Gitter频道]: https://gitter.im/rust-impl-period/WG-libs-guidelines