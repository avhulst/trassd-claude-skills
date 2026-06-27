# trassd-symfony

Skills and agents that enforce **Symfony** best practices: service config &
autowiring, Doctrine entity design and migrations, security voters, and
functional testing.

This is a [Claude Code](https://claude.com/claude-code) plugin. Its skills
trigger automatically when relevant, and its agents become available to the
`Agent` tool.

## Skills

| Skill | Covers |
|-------|--------|
| `symfony-service-config` | Autowiring, autoconfiguration, tags, prefixed parameters, private services |
| `doctrine-entity-design` | Entity attribute mapping, associations, repositories, value objects |
| `doctrine-migrations` | Generating, reviewing, and applying Doctrine migrations safely |
| `symfony-security-voters` | Voters, security expressions, access_control rules |
| `symfony-testing` | WebTestCase/KernelTestCase, DOM crawler, database testing |

## Agents

| Agent | When to use |
|-------|-------------|
| `symfony-code-reviewer` | Review Symfony code against the official best practices. |
| `symfony-security-auditor` | Audit firewalls, access control, voters, password hashing, and CSRF. |

## Installing

This plugin is published through the **trassd** marketplace. Add the marketplace
(by local path or, once published, its git repo), then install:

```
/plugin marketplace add <git-repo-of-the-trassd-marketplace>
/plugin install trassd-symfony@trassd
```

## License

MIT © Andreas van Hulst (see the marketplace `LICENSE`).
