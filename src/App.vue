<template>
  <div class="app-container">
    <h2 class="page-title">日报/周报生成工具</h2>
    <div class="project-management-section">
      <button class="manage-projects-btn" @click="openProjectModal">管理项目</button>
    </div>
    <div class="card-container">
      <!-- 日期卡片按周五、周六、周一、周二、周三、周四顺序排列 -->
      <div v-for="day in days" :key="day.id" class="day-card">
        <div class="card-header">
          <h2 class="day-name">{{ day.name }}</h2>
          <button class="add-task-btn" @click="openAddTaskModal(day.id)">+ 添加任务</button>
        </div>
        <div class="task-list">
          <div v-for="(task, index) in day.tasks" :key="index" class="task-item">
            <div class="task-content">
              <div class="task-main">
                <span class="task-description">{{ task.description }}</span>
              </div>
              <div class="task-meta">
                <span class="task-project">{{ task.project }}</span>
                <span class="task-hours">工作时长: {{ task.workHours }}h</span>
                <span class="task-overtime">加班: {{ task.overtimeHours }}h</span>
              </div>
            </div>
            <div class="task-actions">
              <button @click="openEditTaskModal(day.id, index)">编辑</button>
              <button @click="deleteTask(day.id, index)">删除</button>
            </div>
          </div>
          <div v-if="day.tasks.length === 0" class="empty-task">
            暂无任务，点击"添加任务"开始
          </div>
        </div>
      </div>
    </div>
    <button @click="generateDailyReport" class="generate-report-btn daily-report-btn">生成日报</button>
    <button @click="generateReport" class="generate-report-btn">生成周报</button>
    <button @click="clearAllTasks" class="generate-report-btn clear-tasks-btn">清空所有任务</button>

    <div v-if="dailyReportResult" class="daily-report-result">
      <div v-for="(report, reportIndex) in dailyReportResult" :key="reportIndex" class="daily-report-day">
        <div class="report-header">
          <h2>{{ report.dayName }}日报</h2>
          <button class="copy-btn" @click="copyDailyReport(reportIndex)">复制</button>
        </div>
        <div class="daily-report-section">
          <h3>一、今日计划：</h3>
          <p v-for="plan in report.plans" :key="plan.id">
            {{ plan.id }}. {{ plan.description }}；
          </p>
        </div>
        <div class="daily-report-section">
          <h3>二、完成情况：</h3>
          <p v-for="plan in report.plans" :key="plan.id">
            {{ plan.id }}. 已完成，用时 {{ plan.workHours }} 小时。
          </p>
        </div>
        <div v-if="report.overtime.totalHours > 0" class="daily-report-section">
          <h3>三、加班情况：加班 {{ report.overtime.totalHours }} 小时。</h3>
          <p v-for="task in report.overtime.tasks" :key="task.id">
            {{ task.id }}. {{ task.description }}；加班 {{ task.hours }} 小时。
          </p>
        </div>
      </div>
    </div>

    <div v-if="reportResult" class="report-result">
      <div class="report-header">
        <h2>周报</h2>
        <button class="copy-btn" @click="copyWeeklyReport">复制</button>
      </div>
      <div class="report-projects">
        <div v-for="(project, index) in reportResult.projects" :key="project.name" class="project-section">
          <h3>{{ numberToChinese(index + 1) }}、{{ project.name }}：</h3>
          <p v-for="(task, taskIndex) in project.tasks" :key="taskIndex">
            {{ `${taskIndex + 1}. ${task.description}${(task.workHours > 0 || task.overtimeHours > 0) ? '，' :
              ''}${[task.workHours > 0 ? `工作时长: ${task.workHours} 小时` : '', task.overtimeHours > 0 ? `加班时长:
            ${task.overtimeHours} 小时` : ''].filter(Boolean).join('，')}。` }}
          </p>
        </div>
      </div>
      <div class="report-summary">
        本周总工作时长{{ reportResult.totalWorkHours }} + {{ reportResult.totalOvertimeHours }}小时。
      </div>
    </div>

    <!-- 添加/编辑任务模态框 -->
    <div v-if="isModalOpen" class="modal-overlay">
      <div class="modal-content">
        <h3 class="modal-title">{{ currentTaskId === null ? '添加任务' : '编辑任务' }}</h3>
        <form>
          <div class="form-group">
            <label>任务描述:</label>
            <input type="text" v-model="newTask.description" required>
          </div>
          <div class="form-group">
            <label>项目名称:</label>
            <select v-model="newTask.project" required>
              <option value="" disabled>请选择项目</option>
              <option v-for="project in projects" :key="project" :value="project">
                {{ project }}
              </option>
            </select>
            <input type="text" v-model="newProjectName" v-if="showAddProjectInput" placeholder="输入新项目名称">
            <button type="button" @click="toggleAddProjectInput" v-if="!showAddProjectInput" class="small-btn">+</button>
            <button type="button" @click="addProject" v-if="showAddProjectInput" class="small-btn">确定</button>
          </div>
          <div class="form-row">
            <div class="form-group half">
              <label>工作时长 (小时):</label>
              <input type="number" v-model.number="newTask.workHours" min="0" step="0.5" required>
            </div>
            <div class="form-group half">
              <label>加班时长 (小时):</label>
              <input type="number" v-model.number="newTask.overtimeHours" min="0" step="0.5" required>
            </div>
          </div>
          <div class="modal-buttons">
            <button type="button" @click="closeModal">取消</button>
            <button type="submit" @click="saveTask">保存</button>
          </div>
        </form>
      </div>
    </div>
    
    <!-- 项目管理模态框 -->
    <div v-if="isProjectModalOpen" class="modal-overlay">
      <div class="modal-content">
        <h3 class="modal-title">管理项目</h3>
        <div class="project-list">
          <div v-for="(project, index) in projects" :key="project" class="project-item">
            <span style="color: black;">{{ project }}</span>
            <button type="button" @click="deleteProject(index)" class="delete-project-btn" v-if="projects.length > 1">删除</button>
          </div>
        </div>
        <div class="add-project-form">
          <input type="text" v-model="newProjectName" placeholder="输入新项目名称">
          <button type="button" @click="addProject" class="add-project-btn">添加项目</button>
        </div>
        <div class="modal-buttons">
          <button type="button" @click="closeProjectModal">关闭</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue';

// 定义任务类型
interface Task {
  description: string;
  project: string;
  workHours: number;
  overtimeHours: number;
}

// 从localStorage加载项目列表
const loadProjects = (): string[] => {
  const savedProjects = localStorage.getItem('weeklyProjects');
  if (savedProjects) {
    try {
      return JSON.parse(savedProjects);
    } catch (e) {
      console.error('Failed to parse saved projects', e);
    }
  }
  // 默认项目列表
  return ['默认项目'];
};

// 保存项目列表到localStorage
const saveProjectsToLocalStorage = () => {
  localStorage.setItem('weeklyProjects', JSON.stringify(projects));
};

// 项目列表
const projects = reactive<string[]>(loadProjects());

// 定义日期类型
interface Day {
  id: number;
  name: string;
  tasks: Task[];
}

// 从localStorage加载任务数据
const loadFromLocalStorage = (): Day[] => {
  const savedDays = localStorage.getItem('weeklyTasks');
  if (savedDays) {
    try {
      return JSON.parse(savedDays);
    } catch (e) {
      console.error('Failed to parse saved tasks', e);
    }
  }
  // 默认初始化数据
  return [
    { id: 1, name: '周五', tasks: [] },
    { id: 2, name: '周六', tasks: [] },
    { id: 3, name: '周一', tasks: [] },
    { id: 4, name: '周二', tasks: [] },
    { id: 5, name: '周三', tasks: [] },
    { id: 6, name: '周四', tasks: [] },
  ];
};

// 保存任务数据到localStorage
const saveToLocalStorage = () => {
  localStorage.setItem('weeklyTasks', JSON.stringify(days));
};

// 初始化日期数据
const days = reactive<Day[]>(loadFromLocalStorage());

// 模态框状态
const isModalOpen = ref(false);
const currentDayId = ref<number | null>(null);
const currentTaskId = ref<number | null>(null);

const reportResult = ref<null | {
  totalWorkHours: number;
  totalOvertimeHours: number;
  projects: Array<{
    name: string;
    tasks: Array<{
      index: number;
      description: string;
      workHours: number;
      overtimeHours: number;
    }>;
  }>
}>(null);
const dailyReportResult = ref<DailyReport[] | null>(null);

// 新任务/编辑任务数据
const newTask = reactive<Task>({
  description: '',
  project: '',
  workHours: 0,
  overtimeHours: 0,
});

// 项目模态框状态
const isProjectModalOpen = ref(false);
const newProjectName = ref('');
const showAddProjectInput = ref(false);

// 打开添加任务模态框
function openAddTaskModal(dayId: number) {
  currentDayId.value = dayId;
  currentTaskId.value = null;
  // 重置表单
  newTask.description = '';
  // 默认选中第一个项目
  newTask.project = projects.length > 0 ? projects[0] : '';
  newTask.workHours = 0;
  newTask.overtimeHours = 0;
  isModalOpen.value = true;
  showAddProjectInput.value = false;
}

// 打开编辑任务模态框
function openEditTaskModal(dayId: number, taskIndex: number) {
  currentDayId.value = dayId;
  currentTaskId.value = taskIndex;
  const day = days.find(d => d.id === dayId);
  if (day && day.tasks[taskIndex]) {
    const task = day.tasks[taskIndex];
    // 填充表单
    newTask.description = task.description;
    newTask.project = task.project;
    // 如果项目不存在于列表中，添加它
    if (task.project && !projects.includes(task.project)) {
      projects.push(task.project);
      saveProjectsToLocalStorage();
    }
    newTask.workHours = task.workHours;
    newTask.overtimeHours = task.overtimeHours;
    isModalOpen.value = true;
    showAddProjectInput.value = false;
  }
}

// 关闭模态框
function closeModal() {
  isModalOpen.value = false;
}

// 保存任务
function saveTask() {
  if (!currentDayId.value) return;

  const day = days.find(d => d.id === currentDayId.value);
  if (!day) return;

  if (currentTaskId.value === null) {
    // 添加新任务
    day.tasks.push({
      description: newTask.description,
      project: newTask.project,
      workHours: newTask.workHours,
      overtimeHours: newTask.overtimeHours,
    });
  } else {
    // 编辑现有任务
    day.tasks[currentTaskId.value] = {
      description: newTask.description,
      project: newTask.project,
      workHours: newTask.workHours,
      overtimeHours: newTask.overtimeHours,
    };
  }

  closeModal();
  saveToLocalStorage();
}

// 清空所有任务
import { ElMessageBox, ElMessage } from 'element-plus';

async function clearAllTasks() {
  try {
    await ElMessageBox.confirm(
      '确定要清空所有任务吗？此操作不可恢复。',
      '警告',
      {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      }
    );
    days.forEach(day => {
      day.tasks = [];
    });
    saveToLocalStorage();
    ElMessage.success('所有任务已成功清空');
  } catch {
    ElMessage.info('已取消清空操作');
  }
}

// 删除任务
function deleteTask(dayId: number, taskIndex: number) {
  const day = days.find(d => d.id === dayId);
  if (day) {
    day.tasks.splice(taskIndex, 1);
    saveToLocalStorage();
  }
}

// 切换添加新项目输入框
const toggleAddProjectInput = () => {
  showAddProjectInput.value = !showAddProjectInput.value;
  if (showAddProjectInput.value) {
    newProjectName.value = '';
  }
};

// 添加新项目
const addProject = () => {
  if (newProjectName.value.trim() && !projects.includes(newProjectName.value.trim())) {
    projects.push(newProjectName.value.trim());
    saveProjectsToLocalStorage();
    newProjectName.value = '';
    showAddProjectInput.value = false;
    ElMessage.success('项目添加成功');
  } else if (projects.includes(newProjectName.value.trim())) {
    ElMessage.warning('该项目已存在');
  }
};

// 删除项目
const deleteProject = (index: number) => {
  const projectToDelete = projects[index];
  
  // 检查是否有任务使用该项目
  const hasTasksUsingProject = days.some(day => 
    day.tasks.some(task => task.project === projectToDelete)
  );
  
  if (hasTasksUsingProject) {
    ElMessage.warning('该项目已有任务，无法删除');
    return;
  }
  
  projects.splice(index, 1);
  saveProjectsToLocalStorage();
  ElMessage.success('项目删除成功');
};

// 打开项目管理模态框
const openProjectModal = () => {
  newProjectName.value = '';
  isProjectModalOpen.value = true;
};

// 关闭项目管理模态框
const closeProjectModal = () => {
  isProjectModalOpen.value = false;
};

// 定义日报数据接口
interface DailyReport {
  dayName: string;
  plans: Array<{
    id: number;
    description: string;
    workHours: number;
  }>;
  completions: Array<{
    id: number;
    status: string;
    hours: number;
  }>;
  overtime: {
    totalHours: number;
    tasks: Array<{
      id: number;
      description: string;
      hours: number;
    }>;
  };
}

// 生成日报
function generateDailyReport() {
  // 定义日期顺序：周五、周六、周一、周二、周三、周四
  const dayOrder = ['周五', '周六', '周一', '周二', '周三', '周四'];
  const dailyReports: DailyReport[] = [];

  dayOrder.forEach(dayName => {
    const day = days.find(d => d.name === dayName);
    if (!day || day.tasks.length === 0) return;

    // 过滤任务：排除工作时长为0且加班时长大于0的任务
    const regularTasks = day.tasks.filter(task => !(task.workHours === 0 && task.overtimeHours > 0));

    // 构建日报数据
    const report = {
      dayName,
      plans: regularTasks.map((task, index) => ({
        id: index + 1,
        description: task.description,
        workHours: task.workHours
      })),
      completions: regularTasks.map((task, index) => ({
        id: index + 1,
        status: '完成',
        hours: task.workHours + task.overtimeHours
      })),
      overtime: {
        totalHours: day.tasks.reduce((sum, task) => sum + task.overtimeHours, 0),
        tasks: day.tasks.filter(task => task.overtimeHours > 0).map((task, index) => ({
          id: index + 1,
          description: task.description,
          hours: task.overtimeHours
        }))
      }
    };

    dailyReports.push(report);
  });

  dailyReportResult.value = dailyReports;
}

// 生成周报
function numberToChinese(num: number) {
  const chineseNumbers = ['', '一', '二', '三', '四', '五', '六', '七', '八', '九', '十'];
  return chineseNumbers[num] || num;
}

// 复制日报
function copyDailyReport(reportIndex: number) {
  if (!dailyReportResult.value || !dailyReportResult.value[reportIndex]) return;
  
  const report = dailyReportResult.value[reportIndex];
  let reportText = ``;
  
  // 今日计划 - 只包含非加班任务
  if (report.plans.length > 0) {
    reportText += '一、今日计划：\n';
    report.plans.forEach(plan => {
      reportText += `${plan.id}. ${plan.description}\n`;
    });
  }
  
  // 完成情况 - 只包含非加班任务
  if (report.completions.length > 0) {
    reportText += '二、完成情况：\n';
    report.plans.forEach(plan => {
      reportText += `${plan.id}. 已完成，用时 ${plan.workHours} 小时\n`;
    });
  }
  
  // 加班情况
  if (report.overtime.totalHours > 0) {
    reportText += `三、加班情况：加班 ${report.overtime.totalHours} 小时\n`;
    report.overtime.tasks.forEach(task => {
      reportText += `${task.id}. ${task.description}；加班 ${task.hours} 小时\n`;
    });
  }
  
  // 复制到剪贴板
  navigator.clipboard.writeText(reportText).then(() => {
    ElMessage.success('日报已复制到剪贴板');
  }).catch(() => {
    ElMessage.error('复制失败，请手动复制');
  });
}

// 复制周报
function copyWeeklyReport() {
  if (!reportResult.value) return;
  
  let reportText = '周报\n';
  
  // 按项目分类的任务
  reportResult.value.projects.forEach((project, index) => {
    reportText += `${numberToChinese(index + 1)}、${project.name}：\n`;
    project.tasks.forEach((task, taskIndex) => {
      const hoursText = [];
      if (task.workHours > 0) {
        hoursText.push(`工作时长: ${task.workHours} 小时`);
      }
      if (task.overtimeHours > 0) {
        hoursText.push(`加班时长: ${task.overtimeHours} 小时`);
      }
      reportText += `${taskIndex + 1}. ${task.description}${hoursText.length > 0 ? '，' : ''}${hoursText.join('，')}\n`;
    });
  });
  
  // 周报总结
  reportText += `本周总工作时长${reportResult.value.totalWorkHours} + ${reportResult.value.totalOvertimeHours}小时`;
  
  // 复制到剪贴板
  navigator.clipboard.writeText(reportText).then(() => {
    ElMessage.success('周报已复制到剪贴板');
  }).catch(() => {
    ElMessage.error('复制失败，请手动复制');
  });
}

function generateReport() {
  let totalWorkHours = 0;
  let totalOvertimeHours = 0;
  const projects: Record<string, Array<{
    description: string;
    workHours: number;
    overtimeHours: number;
    index?: number;
  }>> = {};

  // 收集所有任务并按项目分类
  days.forEach(day => {
    day.tasks.forEach(task => {
      const isSaturday = day.name === '周六';
      const workHours = isSaturday ? 0 : task.workHours;
      const overtimeHours = isSaturday ? (task.workHours + task.overtimeHours) : task.overtimeHours;

      totalWorkHours += workHours;
      totalOvertimeHours += overtimeHours;

      if (!projects[task.project]) {
        projects[task.project] = [];
      }
      projects[task.project].push({
        description: task.description,
        workHours: workHours,
        overtimeHours: overtimeHours
      });
    });
  });

  // 为所有任务统一编号
  let taskIndex = 0;
  const projectList = Object.entries(projects).map(([name, tasks]) => {
    tasks.forEach(task => {
      taskIndex++;
      task.index = taskIndex;
    });
    return { name, tasks };
  });

  // 更新报告结果
  // 修复类型不匹配问题
  reportResult.value = {
    totalWorkHours,
    totalOvertimeHours,
    projects: projectList.map(project => ({
      name: project.name,
      tasks: project.tasks.map(task => ({
        index: task.index || 0,
        description: task.description,
        workHours: task.workHours,
        overtimeHours: task.overtimeHours
      }))
    }))
  } as typeof reportResult.value;
}
</script>

<style>
/* 统一的颜色主题变量 */
:root {
  --primary-color: #42B983;
  --primary-light: #e6f7ef;
  --primary-dark: #359469;
  --text-on-primary: white;
  --text-primary: #333333;
  --text-secondary: #666666;
  --text-muted: #999999;
  --secondary-color: #3498db;
  --background-light: #f5f5f5;
}
</style>

<style scoped>
.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Arial', sans-serif;
}

.page-title {
  text-align: center;
  color: #42B983;
  margin-bottom: 30px;
}

.card-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 20px;
  margin-bottom: 40px;
}

.day-card {
  background-color: white;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  transition: transform 0.2s;
}

.day-card:hover {
    transform: translateY(-5px);
  }
  
  .report-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 15px;
  }
  
  .copy-btn {
    padding: 6px 16px;
    background-color: var(--secondary-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    transition: background-color 0.3s;
  }
  
  .copy-btn:hover {
    background-color: #2980b9;
  }

  /* 项目管理样式 */
  .project-management-section {
    margin-bottom: 20px;
    text-align: right;
  }
  
  .manage-projects-btn {
    padding: 8px 16px;
    background-color: var(--primary-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
  }
  
  .manage-projects-btn:hover {
    background-color: var(--primary-dark);
  }
  
  .small-btn {
    padding: 4px 8px;
    margin-left: 8px;
    background-color: var(--secondary-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 12px;
  }
  
  .small-btn:hover {
    background-color: #2980b9;
  }
  
  .project-list {
    margin: 20px 0;
    max-height: 200px;
    overflow-y: auto;
  }
  
  .project-item {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 10px;
    border-bottom: 1px solid #eee;
  }
  
  .delete-project-btn {
    padding: 4px 12px;
    background-color: #e74c3c;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 12px;
  }
  
  .delete-project-btn:hover {
    background-color: #c0392b;
  }
  
  .add-project-form {
    display: flex;
    gap: 10px;
    margin: 20px 0;
  }
  
  .add-project-form input {
    flex: 1;
    padding: 8px;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
  
  .add-project-btn {
    padding: 8px 16px;
    background-color: var(--secondary-color);
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 14px;
    transition: background-color 0.3s;
  }
  
  .add-project-btn:hover {
    background-color: #2980b9;
  }
  
  .card-header {
  background-color: var(--primary-color);
  color: var(--text-on-primary);
  padding: 15px 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.day-name {
  margin: 0;
  font-size: 1.2rem;
}

.add-task-btn {
  background-color: white;
  color: #42b983;
  border: none;
  border-radius: 20px;
  padding: 5px 15px;
  cursor: pointer;
  font-weight: bold;
}

.task-list {
  padding: 15px;
  max-height: 400px;
  overflow-y: auto;
}

.task-item {
  background-color: #f9f9f9;
  border-radius: 8px;
  padding: 12px;
  margin-bottom: 10px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.task-content {
  flex: 1;
}

.task-main {
  margin-bottom: 5px;
}

.task-description {
  font-size: 1rem;
  color: #333;
}

.task-meta {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  font-size: 0.8rem;
  color: var(--text-secondary);
}

.task-project {
  background-color: var(--primary-light);
  padding: 2px 8px;
  border-radius: 12px;
  color: var(--primary-dark);
}

.task-actions {
  display: flex;
  gap: 8px;
}

.task-actions button {
  background: none;
  border: none;
  cursor: pointer;
  color: var(--primary-color);
}

.task-actions button:last-child {
  color: #e74c3c;
}

.empty-task {
  text-align: center;
  padding: 20px;
  color: var(--text-muted);
  font-style: italic;
}

.generate-btn {
  background-color: var(--primary-color);
  color: var(--text-on-primary);
  border: none;
  border-radius: 5px;
  padding: 12px 30px;
  font-size: 1rem;
  cursor: pointer;
  display: inline-block;
  margin-right: 10px;
  margin-top: 20px;
}

.daily-report-btn {
  margin-left: 10px;
  background-color: var(--secondary-color);
}

.clear-tasks-btn {
  background-color: #ff4444;
  margin-left: 10px;
}

.clear-tasks-btn:hover {
  background-color: #cc0000;
}

.generate-report-btn {
  background-color: var(--primary-color);
  color: var(--text-on-primary);
  border: none;
  border-radius: 5px;
  padding: 12px 30px;
  font-size: 1rem;
  cursor: pointer;
  display: inline-block;
  margin-right: 10px;
  margin-top: 20px;
}

.daily-report-result {
  margin-top: 20px;
  padding: 15px;
  background-color: var(--primary-light);
  border-radius: 8px;
  color: var(--text-on-primary);
}

.daily-report-day {
  margin-bottom: 25px;
  padding: 15px;
  background-color: white;
  border-radius: 8px;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.05);
  color: var(--text-primary);
  line-height: 1.6;
}

.daily-report-day h2 {
  color: var(--primary-color);
  margin-top: 0;
  margin-bottom: 15px;
}

.daily-report-day:last-child {
  border-bottom: none;
}

.daily-report-section {
  margin-bottom: 15px;
}

.daily-report-section h3 {
  color: var(--primary-dark);
  border-bottom: 1px solid var(--primary-light);
  padding-bottom: 5px;
  margin-top: 15px;
  margin-bottom: 10px;
}

.daily-report-section ul {
  margin-left: 20px;
}

.daily-report-section li {
  margin-bottom: 5px;
  color: var(--text-primary);
  line-height: 1.5;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background-color: white;
  border-radius: 10px;
  width: 90%;
  max-width: 500px;
  padding: 20px;
}

.modal-title {
  margin-top: 0;
  color: var(--text-primary);
}

.form-group {
  margin-bottom: 15px;
}

.form-row {
  display: flex;
  gap: 10px;
}

.half {
  flex: 1;
}

label {
  display: block;
  margin-bottom: 5px;
  color: var(--text-secondary);
}

input {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.modal-buttons {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 20px;
}

.modal-buttons button {
  padding: 8px 15px;
  border-radius: 4px;
  cursor: pointer;
}

.modal-buttons button:first-child {
  background-color: #f2f2f2;
  border: 1px solid #ddd;
}

.modal-buttons button:last-child {
  background-color: var(--primary-color);
  color: var(--text-on-primary);
  border: none;
}

.report-result {
  margin-top: 20px;
  padding: 20px;
  background-color: var(--primary-color);
  border-radius: 8px;
  color: var(--text-on-primary);
}

.report-summary {
  font-weight: bold;
  margin-bottom: 15px;
  color: white;
}

.report-projects {
  margin-top: 15px;
}

.project-section {
  margin-bottom: 15px;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.project-section h3 {
  margin-bottom: 5px;
  color: white;
  font-size: 1.1rem;
}

.project-section ul {
  padding-left: 20px;
  margin-top: 8px;
}

.project-section li {
  margin-bottom: 5px;
  color: var(--primary-light);
  line-height: 1.5;
}
</style>