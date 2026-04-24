# Task for ideator

We're designing a custom Agno SkillLoader for a production application. The stack is MongoDB (source of truth for skills) + Redis (cache layer). A custom SkillLoader should try Redis first, fall back to MongoDB, and populate Redis on miss.

## Agno SkillLoader contract
```python
class SkillLoader(ABC):
    @abstractmethod
    def load(self) -> List[Skill]:
        pass
```

## Skill dataclass (key fields)
```python
@dataclass
class Skill:
    name: str
    description: str
    instructions: str
    source_path: str          # agent_skills.py reads scripts/references from this local path
    scripts: List[str]        # list of filenames
    references: List[str]     # list of filenames
```

## The source_path problem
agent_skills.py does:
```python
refs_dir = Path(skill.source_path) / "references"
scripts_dir = Path(skill.source_path) / "scripts"
content = read_file_safe(ref_file)  # reads from local disk
```
So scripts/references content must exist on local disk, even if they come from DB/cache.

## Questions to explore
1. What does the MongoDB schema look like for skills? (inline content vs references)
2. What goes in Redis — full skill content, or just metadata/pointers?
3. What's the cache key strategy?
4. What's the TTL strategy — and is TTL even the right invalidation model?
5. How does the loader handle the source_path materialization (writing to local disk)?
6. What's the cache invalidation strategy when a skill is updated in MongoDB?
7. Should Redis cache the full materialized skill or the raw DB document?
8. What happens on Redis miss vs MongoDB miss?
9. Race conditions — what if two instances materialize the same skill simultaneously?
10. Is a two-layer cache (Redis + local in-process dict) worth it?

Push on the real trade-offs. What's the cleanest implementation of this MongoSkillLoader with Redis cache?