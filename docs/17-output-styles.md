## 十七、输出样式系统

### 17.1 输出样式加载

```typescript
// outputStyles/loadOutputStylesDir.ts
export type OutputStyleConfig = {
  name: string;
  description: string;
  prompt: string;
  source: string;
  keepCodingInstructions: boolean;
};

export function loadOutputStyles(): OutputStyleConfig[] {
  const styles: OutputStyleConfig[] = [];
  
  // 项目级样式
  const projectDir = join(getProjectDir(), '.claude', 'output-styles');
  styles.push(...loadMarkdownFilesForSubdir(projectDir, 'project'));
  
  // 用户级样式
  const userDir = join(getHomeDir(), '.claude', 'output-styles');
  styles.push(...loadMarkdownFilesForSubdir(userDir, 'user'));
  
  return styles;
}

// Markdown 解析
function parseOutputStyle(
  filePath: string,
  content: string,
  source: 'project' | 'user',
): OutputStyleConfig {
  const { frontmatter, body } = parseFrontmatter(content);
  
  return {
    name: frontmatter.name ?? basename(filePath, '.md'),
    description: frontmatter.description ??
      extractDescriptionFromMarkdown(body),
    prompt: body,
    source: filePath,
    keepCodingInstructions: frontmatter['keep-coding-instructions'] !== 'false',
  };
}

// 缓存管理
const loadOutputStylesDirStyles = memoize(loadOutputStyles);

export function clearOutputStyleCaches(): void {
  loadOutputStylesDirStyles.cache.clear?.();
}
```

---
