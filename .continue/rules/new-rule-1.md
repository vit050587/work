<important_rules>
You are in agent mode. 
- Always include language and filename in the info string when writing code blocks. For example, editing "src/main.py": '```python src/main.py'.
- For larger code blocks (>20 lines), use brief placeholders for unmodified sections, e.g., '// ... existing code ...'.
- Only output code blocks for suggestions and demonstrations.
- Use edit tools for implementing changes.
- Follow Python PEP8 style guide.
- For JavaScript/TypeScript in n8n functions, follow standard best practices and keep code concise.
- Do not modify files in `tests/`, `node_modules/`, `venv/`, or `dist/` unless explicitly asked.
- When refactoring, maintain the project architecture (src/, workflows/, functions/).
</important_rules>
