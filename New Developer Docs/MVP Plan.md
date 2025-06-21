# Atelier MVP - Detailed Sprint Plans

## Week 1: Foundation & GitHub Integration
**Sprint Goal**: Establish secure GitHub connection with real-time repository monitoring
**Deliverable**: Connected repositories with webhook-based change detection

### Day 1-2: Project Architecture & Database Design
**Focus**: Foundational decisions that will support 100+ repositories

#### Technical Tasks
```bash
# Project structure (monorepo approach)
atelier/
├── apps/
│   ├── web/           # Next.js frontend
│   ├── api/           # Node.js backend
│   └── webhook/       # Dedicated webhook handler
├── packages/
│   ├── database/      # Prisma schema & migrations
│   ├── github/        # GitHub API integration
│   └── analysis/      # Code analysis engine
└── tools/
    ├── dev/           # Development scripts
    └── deploy/        # Deployment configuration
```

#### Database Schema Design
```prisma
// Core entities optimized for GitHub integration
model User {
  id          String   @id @default(cuid())
  githubId    Int      @unique
  username    String   @unique
  email       String?
  avatar      String?
  accessToken String   @encrypted
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  repositories Repository[]
  analyses     Analysis[]
}

model Repository {
  id          String   @id @default(cuid())
  githubId    Int      @unique
  name        String
  fullName    String   @unique
  private     Boolean
  language    String?
  framework   String?
  webhookId   String?
  lastAnalysis DateTime?
  
  userId      String
  user        User @relation(fields: [userId], references: [id])
  
  analyses    Analysis[]
  insights    Insight[]
  
  @@index([userId, lastAnalysis])
}

model Analysis {
  id          String   @id @default(cuid())
  commitSha   String
  patterns    Json     // Detected architectural patterns
  metrics     Json     // Complexity, dependencies, etc.
  status      AnalysisStatus
  processingTime Int?  // milliseconds
  createdAt   DateTime @default(now())
  
  repositoryId String
  repository   Repository @relation(fields: [repositoryId], references: [id])
  
  userId      String
  user        User @relation(fields: [userId], references: [id])
  
  insights    Insight[]
  
  @@index([repositoryId, createdAt])
  @@index([commitSha])
}

enum AnalysisStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
}
```

#### Architectural Learning Points
- **Database Design**: ACID compliance vs. NoSQL flexibility for repository data
- **Encryption Strategy**: Token security patterns and key rotation
- **Indexing Strategy**: Query optimization for repository searches and analysis history

### Day 3-4: GitHub OAuth & API Integration
**Focus**: Secure, scalable GitHub authentication and API usage

#### Implementation Code Sample
```typescript
// packages/github/src/auth.ts
import { Octokit } from '@octokit/rest';
import { createAppAuth } from '@octokit/auth-app';

export class GitHubService {
  private octokit: Octokit;
  
  constructor(private accessToken: string) {
    this.octokit = new Octokit({
      auth: accessToken,
      throttle: {
        onRateLimit: (retryAfter, options) => {
          console.warn(`Rate limit exceeded, retrying after ${retryAfter}s`);
          return true; // Retry once
        },
        onSecondaryRateLimit: () => true
      }
    });
  }
  
  async getRepositories(page = 1, perPage = 30) {
    try {
      const response = await this.octokit.repos.listForAuthenticatedUser({
        page,
        per_page: perPage,
        sort: 'updated',
        affiliation: 'owner,collaborator'
      });
      
      return response.data.map(repo => ({
        githubId: repo.id,
        name: repo.name,
        fullName: repo.full_name,
        private: repo.private,
        language: repo.language,
        updatedAt: repo.updated_at
      }));
    } catch (error) {
      throw new GitHubAPIError('Failed to fetch repositories', error);
    }
  }
  
  async createWebhook(repoFullName: string, webhookUrl: string) {
    const [owner, repo] = repoFullName.split('/');
    
    try {
      const response = await this.octokit.repos.createWebhook({
        owner,
        repo,
        config: {
          url: webhookUrl,
          content_type: 'json',
          secret: process.env.WEBHOOK_SECRET
        },
        events: ['push', 'pull_request']
      });
      
      return response.data.id.toString();
    } catch (error) {
      throw new GitHubAPIError('Failed to create webhook', error);
    }
  }
}
```

#### Rate Limiting Strategy
```typescript
// packages/github/src/rate-limiter.ts
export class GitHubRateLimiter {
  private static instance: GitHubRateLimiter;
  private requestQueue: Map<string, Date[]> = new Map();
  
  static getInstance() {
    if (!this.instance) {
      this.instance = new GitHubRateLimiter();
    }
    return this.instance;
  }
  
  async checkRateLimit(userId: string): Promise<boolean> {
    const userRequests = this.requestQueue.get(userId) || [];
    const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
    
    // Clean old requests
    const recentRequests = userRequests.filter(time => time > oneHourAgo);
    
    if (recentRequests.length >= 4500) { // Leave buffer for GitHub's 5000/hour
      return false;
    }
    
    recentRequests.push(new Date());
    this.requestQueue.set(userId, recentRequests);
    return true;
  }
}
```

### Day 5-6: Webhook Infrastructure
**Focus**: Real-time repository change detection and processing

#### Webhook Handler Implementation
```typescript
// apps/webhook/src/handler.ts
import { verify } from 'crypto';
import { queue } from '@atelier/queue';

export async function handleWebhook(req: Request) {
  // Verify GitHub signature
  const signature = req.headers.get('x-hub-signature-256');
  const payload = await req.text();
  
  if (!verifyGitHubSignature(payload, signature)) {
    return new Response('Unauthorized', { status: 401 });
  }
  
  const event = JSON.parse(payload);
  const eventType = req.headers.get('x-github-event');
  
  switch (eventType) {
    case 'push':
      await queue.add('analyze-repository', {
        repositoryId: event.repository.id,
        commitSha: event.head_commit.id,
        changedFiles: event.head_commit.modified.concat(event.head_commit.added)
      });
      break;
      
    case 'pull_request':
      if (event.action === 'opened' || event.action === 'synchronize') {
        await queue.add('analyze-pull-request', {
          repositoryId: event.repository.id,
          prNumber: event.pull_request.number,
          headSha: event.pull_request.head.sha
        });
      }
      break;
  }
  
  return new Response('OK');
}

function verifyGitHubSignature(payload: string, signature: string): boolean {
  const secret = process.env.WEBHOOK_SECRET;
  const expectedSignature = `sha256=${crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex')}`;
  
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expectedSignature)
  );
}
```

### Day 7: Local Development Environment
**Focus**: Streamlined development setup and testing

#### Development Configuration
```yaml
# docker-compose.dev.yml
version: '3.8'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: atelier_dev
      POSTGRES_USER: atelier
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  webhook-tunnel:
    image: cloudflare/cloudflared:latest
    command: tunnel --url http://localhost:3001
    depends_on:
      - webhook
    
volumes:
  postgres_data:
```

```typescript
// tools/dev/setup.ts
export async function setupDevelopment() {
  console.log('🚀 Setting up Atelier development environment...');
  
  // 1. Check dependencies
  await checkDependencies(['node', 'docker', 'git']);
  
  // 2. Start services
  await exec('docker-compose -f docker-compose.dev.yml up -d');
  
  // 3. Run migrations
  await exec('npx prisma migrate dev');
  
  // 4. Seed database
  await seedDatabase();
  
  // 5. Start development servers
  await Promise.all([
    exec('npm run dev:web'),
    exec('npm run dev:api'),
    exec('npm run dev:webhook')
  ]);
  
  console.log('✅ Development environment ready!');
  console.log('🌐 Web: http://localhost:3000');
  console.log('🔗 API: http://localhost:3001');
  console.log('📨 Webhooks: https://tunnel-url.trycloudflare.com');
}
```

**Week 1 Success Criteria:**
- ✅ GitHub OAuth authentication working
- ✅ Repository list displayed in dashboard
- ✅ Webhooks successfully receiving push events
- ✅ Database schema supporting 1000+ repositories
- ✅ Development environment setup in <5 minutes

---

## Week 2: Analysis Engine Core
**Sprint Goal**: Build pattern recognition system for JavaScript/TypeScript projects
**Deliverable**: Automated architectural insights from repository analysis

### Day 8-9: Code Analysis Pipeline Architecture
**Focus**: Scalable, queue-based processing system

#### Queue Architecture
```typescript
// packages/queue/src/analysis-queue.ts
import Bull from 'bull';
import { Redis } from 'ioredis';

export class AnalysisQueue {
  private queue: Bull.Queue;
  private redis: Redis;
  
  constructor() {
    this.redis = new Redis(process.env.REDIS_URL);
    this.queue = new Bull('repository-analysis', {
      redis: { port: 6379, host: 'localhost' },
      defaultJobOptions: {
        removeOnComplete: 100,
        removeOnFail: 50,
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 2000
        }
      }
    });
    
    this.setupProcessors();
  }
  
  private setupProcessors() {
    this.queue.process('analyze-repository', 5, async (job) => {
      const { repositoryId, commitSha, changedFiles } = job.data;
      
      try {
        const analyzer = new RepositoryAnalyzer(repositoryId);
        const analysis = await analyzer.analyze(commitSha, changedFiles);
        
        await this.saveAnalysisResults(analysis);
        await this.generateInsights(analysis);
        
        return analysis;
      } catch (error) {
        throw new Error(`Analysis failed: ${error.message}`);
      }
    });
  }
  
  async addAnalysisJob(data: AnalysisJobData) {
    return this.queue.add('analyze-repository', data, {
      priority: data.urgent ? 10 : 5,
      delay: data.delay || 0
    });
  }
}
```

### Day 10-11: AST Parsing & Pattern Recognition
**Focus**: Intelligent code analysis for architectural patterns

#### AST Analysis Implementation
```typescript
// packages/analysis/src/ast-analyzer.ts
import * as ts from 'typescript';
import { parse } from '@babel/parser';
import traverse from '@babel/traverse';

export class ASTAnalyzer {
  async analyzeTypeScript(code: string, filePath: string): Promise<FileAnalysis> {
    const sourceFile = ts.createSourceFile(
      filePath,
      code,
      ts.ScriptTarget.Latest,
      true
    );
    
    const analysis: FileAnalysis = {
      components: [],
      imports: [],
      exports: [],
      patterns: [],
      complexity: 0
    };
    
    const visit = (node: ts.Node) => {
      switch (node.kind) {
        case ts.SyntaxKind.FunctionDeclaration:
        case ts.SyntaxKind.ArrowFunction:
          analysis.complexity += this.calculateComplexity(node);
          break;
          
        case ts.SyntaxKind.ImportDeclaration:
          analysis.imports.push(this.extractImport(node as ts.ImportDeclaration));
          break;
          
        case ts.SyntaxKind.ClassDeclaration:
          analysis.components.push(this.extractClass(node as ts.ClassDeclaration));
          break;
      }
      
      ts.forEachChild(node, visit);
    };
    
    visit(sourceFile);
    
    // Detect architectural patterns
    analysis.patterns = this.detectPatterns(analysis);
    
    return analysis;
  }
  
  private detectPatterns(analysis: FileAnalysis): ArchitecturalPattern[] {
    const patterns: ArchitecturalPattern[] = [];
    
    // React Component Pattern
    if (this.isReactComponent(analysis)) {
      patterns.push({
        type: 'REACT_COMPONENT',
        confidence: 0.9,
        description: 'React functional or class component detected',
        suggestions: this.getReactSuggestions(analysis)
      });
    }
    
    // Custom Hook Pattern
    if (this.isCustomHook(analysis)) {
      patterns.push({
        type: 'CUSTOM_HOOK',
        confidence: 0.85,
        description: 'React custom hook pattern detected',
        suggestions: ['Consider memoization', 'Add error handling']
      });
    }
    
    // Repository Pattern
    if (this.isRepositoryPattern(analysis)) {
      patterns.push({
        type: 'REPOSITORY_PATTERN',
        confidence: 0.8,
        description: 'Data access repository pattern detected',
        suggestions: ['Add interface abstraction', 'Consider dependency injection']
      });
    }
    
    return patterns;
  }
}
```

#### Pattern Detection Logic
```typescript
// packages/analysis/src/pattern-detector.ts
export class PatternDetector {
  detectReactPatterns(fileAnalyses: FileAnalysis[]): ProjectPattern[] {
    const patterns: ProjectPattern[] = [];
    
    // Component Architecture Analysis
    const components = fileAnalyses.flatMap(f => f.components);
    const componentStructure = this.analyzeComponentStructure(components);
    
    if (componentStructure.hasAtomicDesign) {
      patterns.push({
        type: 'ATOMIC_DESIGN',
        scope: 'PROJECT',
        confidence: 0.9,
        impact: 'HIGH',
        description: 'Atomic design pattern detected in component structure',
        files: componentStructure.relatedFiles,
        suggestions: [
          'Ensure consistent naming conventions',
          'Consider Storybook for component documentation',
          'Add prop type validation'
        ]
      });
    }
    
    // State Management Patterns
    const stateManagement = this.analyzeStateManagement(fileAnalyses);
    if (stateManagement.pattern === 'REDUX') {
      patterns.push({
        type: 'REDUX_PATTERN',
        scope: 'PROJECT',
        confidence: stateManagement.confidence,
        impact: 'HIGH',
        description: 'Redux state management pattern detected',
        suggestions: [
          'Consider Redux Toolkit for simpler syntax',
          'Implement proper error handling in reducers',
          'Add middleware for async actions'
        ]
      });
    }
    
    return patterns;
  }
  
  private analyzeComponentStructure(components: ComponentInfo[]): ComponentStructure {
    const directories = components.map(c => c.filePath.split('/').slice(0, -1));
    const hasAtomicStructure = this.hasAtomicDesignStructure(directories);
    
    return {
      hasAtomicDesign: hasAtomicStructure,
      componentCount: components.length,
      averageComplexity: components.reduce((sum, c) => sum + c.complexity, 0) / components.length,
      relatedFiles: components.map(c => c.filePath)
    };
  }
}
```

### Day 12-13: AI Integration for Insights
**Focus**: Generate actionable architectural guidance

#### AI Insight Generation
```typescript
// packages/analysis/src/insight-generator.ts
import OpenAI from 'openai';

export class InsightGenerator {
  private openai: OpenAI;
  
  constructor() {
    this.openai = new OpenAI({
      apiKey: process.env.OPENAI_API_KEY
    });
  }
  
  async generateInsights(analysis: ProjectAnalysis): Promise<ArchitecturalInsight[]> {
    const insights: ArchitecturalInsight[] = [];
    
    // Generate pattern-specific insights
    for (const pattern of analysis.patterns) {
      const insight = await this.generatePatternInsight(pattern, analysis);
      if (insight) insights.push(insight);
    }
    
    // Generate complexity insights
    const complexityInsight = await this.generateComplexityInsight(analysis);
    if (complexityInsight) insights.push(complexityInsight);
    
    // Generate dependency insights
    const dependencyInsight = await this.generateDependencyInsight(analysis);
    if (dependencyInsight) insights.push(dependencyInsight);
    
    return insights;
  }
  
  private async generatePatternInsight(
    pattern: ArchitecturalPattern,
    analysis: ProjectAnalysis
  ): Promise<ArchitecturalInsight | null> {
    const prompt = `
    Analyze this architectural pattern in a JavaScript/TypeScript project:
    
    Pattern: ${pattern.type}
    Confidence: ${pattern.confidence}
    Description: ${pattern.description}
    
    Project Context:
    - Total files: ${analysis.fileCount}
    - Main framework: ${analysis.framework}
    - Average complexity: ${analysis.averageComplexity}
    
    Provide a concise architectural insight (max 150 words) that includes:
    1. What this pattern indicates about the project's architecture
    2. One specific improvement suggestion
    3. Potential risks if not addressed
    
    Focus on actionable guidance for intermediate developers.
    `;
    
    try {
      const response = await this.openai.chat.completions.create({
        model: 'gpt-4',
        messages: [{ role: 'user', content: prompt }],
        max_tokens: 200,
        temperature: 0.3
      });
      
      return {
        id: generateId(),
        type: 'PATTERN_ANALYSIS',
        title: `${pattern.type} Analysis`,
        content: response.choices[0].message.content,
        priority: this.calculatePriority(pattern),
        actionable: true,
        relatedFiles: pattern.files || [],
        createdAt: new Date()
      };
    } catch (error) {
      console.error('Failed to generate AI insight:', error);
      return null;
    }
  }
}
```

**Week 2 Success Criteria:**
- ✅ Repository analysis processing within 30 seconds
- ✅ 5+ architectural patterns accurately detected per repository
- ✅ AI-generated insights relevant and actionable (80%+ user rating)
- ✅ Queue system handling 100+ analysis jobs without blocking

---

## Week 3: Visualization & Dashboard
**Sprint Goal**: Create rich, interactive visualizations and user dashboard
**Deliverable**: Real-time dashboard with dependency graphs and architectural insights

### Day 15-16: Frontend Architecture & Component Design
**Focus**: Scalable, reusable component architecture

#### Component Architecture
```typescript
// apps/web/src/components/architecture/VisualizationProvider.tsx
import React, { createContext, useContext, useReducer } from 'react';

interface VisualizationState {
  repositories: Repository[];
  selectedRepo: Repository | null;
  currentAnalysis: Analysis | null;
  visualizations: {
    dependencyGraph: DependencyGraph | null;
    complexityHeatmap: ComplexityData | null;
    architectureOverview: ArchitectureData | null;
  };
  loading: {
    repositories: boolean;
    analysis: boolean;
    visualizations: boolean;
  };
}

export const VisualizationProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(visualizationReducer, initialState);
  
  const actions = {
    selectRepository: (repo: Repository) => {
      dispatch({ type: 'SELECT_REPOSITORY', payload: repo });
      // Trigger analysis loading
      loadRepositoryAnalysis(repo.id);
    },
    
    updateAnalysis: (analysis: Analysis) => {
      dispatch({ type: 'UPDATE_ANALYSIS', payload: analysis });
      // Generate visualizations
      generateVisualizations(analysis);
    }
  };
  
  return (
    <VisualizationContext.Provider value={{ state, actions }}>
      {children}
    </VisualizationContext.Provider>
  );
};
```

#### Reusable Visualization Components
```typescript
// apps/web/src/components/visualizations/DependencyGraph.tsx
import React, { useEffect, useRef } from 'react';
import * as d3 from 'd3';

interface DependencyGraphProps {
  data: DependencyData;
  width?: number;
  height?: number;
  onNodeClick?: (node: DependencyNode) => void;
}

export const DependencyGraph: React.FC<DependencyGraphProps> = ({
  data,
  width = 800,
  height = 600,
  onNodeClick
}) => {
  const svgRef = useRef<SVGSVGElement>(null);
  
  useEffect(() => {
    if (!svgRef.current || !data.nodes.length) return;
    
    const svg = d3.select(svgRef.current);
    svg.selectAll('*').remove(); // Clear previous render
    
    // Create force simulation
    const simulation = d3.forceSimulation(data.nodes)
      .force('link', d3.forceLink(data.links).id(d => d.id).distance(100))
      .force('charge', d3.forceManyBody().strength(-300))
      .force('center', d3.forceCenter(width / 2, height / 2))
      .force('collision', d3.forceCollide().radius(20));
    
    // Create links
    const link = svg.append('g')
      .selectAll('line')
      .data(data.links)
      .join('line')
      .attr('class', 'dependency-link')
      .attr('stroke', '#999')
      .attr('stroke-opacity', 0.6)
      .attr('stroke-width', d => Math.sqrt(d.weight || 1));
    
    // Create nodes
    const node = svg.append('g')
      .selectAll('circle')
      .data(data.nodes)
      .join('circle')
      .attr('class', 'dependency-node')
      .attr('r', d => 5 + (d.importance || 0) * 10)
      .attr('fill', d => getNodeColor(d.type))
      .call(d3.drag()
        .on('start', dragStarted)
        .on('drag', dragged)
        .on('end', dragEnded))
      .on('click', (event, d) => {
        onNodeClick?.(d);
      });
    
    // Add labels
    const label = svg.append('g')
      .selectAll('text')
      .data(data.nodes)
      .join('text')
      .attr('class', 'node-label')
      .text(d => d.name)
      .attr('font-size', '12px')
      .attr('dx', 15)
      .attr('dy', 4);
    
    // Update positions on simulation tick
    simulation.on('tick', () => {
      link
        .attr('x1', d => d.source.x)
        .attr('y1', d => d.source.y)
        .attr('x2', d => d.target.x)
        .attr('y2', d => d.target.y);
      
      node
        .attr('cx', d => d.x)
        .attr('cy', d => d.y);
      
      label
        .attr('x', d => d.x)
        .attr('y', d => d.y);
    });
    
    function dragStarted(event, d) {
      if (!event.active) simulation.alphaTarget(0.3).restart();
      d.fx = d.x;
      d.fy = d.y;
    }
    
    function dragged(event, d) {
      d.fx = event.x;
      d.fy = event.y;
    }
    
    function dragEnded(event, d) {
      if (!event.active) simulation.alphaTarget(0);
      d.fx = null;
      d.fy = null;
    }
    
  }, [data, width, height]);
  
  return (
    <div className="dependency-graph">
      <svg ref={svgRef} width={width} height={height} />
    </div>
  );
};
```

### Day 17-18: Real-time Data Synchronization
**Focus**: WebSocket integration for live updates

#### Real-time Updates Implementation
```typescript
// apps/web/src/hooks/useRealtimeAnalysis.ts
import { useEffect, useState } from 'react';
import { io, Socket } from 'socket.io-client';

export function useRealtimeAnalysis(repositoryId: string) {
  const [socket, setSocket] = useState<Socket | null>(null);
  const [analysis, setAnalysis] = useState<Analysis | null>(null);
  const [isConnected, setIsConnected] = useState(false);
  
  useEffect(() => {
    const newSocket = io(process.env.NEXT_PUBLIC_WS_URL, {
      auth: {
        token: localStorage.getItem('auth_token')
      }
    });
    
    newSocket.on('connect', () => {
      setIsConnected(true);
      newSocket.emit('subscribe', { repositoryId });
    });
    
    newSocket.on('disconnect', () => {
      setIsConnected(false);
    });
    
    newSocket.on('analysis_started', (data: { repositoryId: string, commitSha: string }) => {
      if (data.repositoryId === repositoryId) {
        setAnalysis(prev => ({ ...prev, status: 'PROCESSING' }));
      }
    });
    
    newSocket.on('analysis_completed', (data: Analysis) => {
      if (data.repositoryId === repositoryId) {
        setAnalysis(data);
      }
    });
    
    newSocket.on('insight_generated', (insight: ArchitecturalInsight) => {
      // Update UI with new insight
      setAnalysis(prev => prev ? {
        ...prev,
        insights: [...prev.insights, insight]
      } : null);
    });
    
    setSocket(newSocket);
    
    return () => {
      newSocket.close();
    };
  }, [repositoryId]);
  
  return {
    analysis,
    isConnected,
    socket
  };
}
```

### Day 19-20: Dashboard Layout & Performance
**Focus**: Optimized dashboard with efficient rendering

#### Performance-Optimized Dashboard
```typescript
// apps/web/src/pages/dashboard/[repositoryId].tsx
import React, { useMemo } from 'react';
import { useQuery } from '@tanstack/react-query';
import { Suspense, lazy } from 'react';

// Lazy load heavy visualization components
const DependencyGraph = lazy(() => import('@/components/visualizations/DependencyGraph'));
const ComplexityHeatmap = lazy(() => import('@/components/visualizations/ComplexityHeatmap'));
const ArchitectureOverview = lazy(() => import('@/components/visualizations/ArchitectureOverview'));

export default function RepositoryDashboard({ repositoryId }: { repositoryId: string }) {
  const { data: repository, isLoading } = useQuery({
    queryKey: ['repository', repositoryId],
    queryFn: () => fetchRepository(repositoryId),
    staleTime: 5 * 60 * 1000 // 5 minutes
  });
  
  const { data: analysis } = useQuery({
    queryKey: ['analysis', repositoryId],
    queryFn: () => fetchLatestAnalysis(repositoryId),
    refetchInterval: 30000, // Refetch every 30s
    enabled: !!repository
  });
  
  // Memoize expensive calculations
  const dashboardData = useMemo(() => {
    if (!analysis) return null;
    
    return {
      healthScore: calculateHealthScore(analysis),
      criticalIssues: analysis.insights.filter(i => i.priority === 'HIGH'),
      recentChanges: analysis.recentChanges.slice(0, 10),
      dependencies: processDependencyData(analysis.dependencies)
    };
  }, [analysis]);
  
  if (isLoading) {
    return <DashboardSkeleton />;
  }
  
  return (
    <div className="dashboard-grid">
      {/* Health