---
title: SoloX之报告导出
date: 2023-07-14 08:41:05
tags: 测试开发 性能 工具
categories: 测试开发
---
## excel

```python
@api.route('/apm/export/report', methods=['post', 'get'])
def exportReport():
    platform = method._request(request, 'platform')
    scene = method._request(request, 'scene')
    try:
        path = f.export_excel(platform=platform, scene=scene)
        result = {'status': 1, 'msg':'success', 'path': path}
    except Exception as e:
        traceback.print_exc()
        result = {'status': 0, 'msg':str(e)}    
    return result
```

```python
def export_excel(self, platform, scene):
    logger.info('Exporting excel ...')
    android_log_file_list = ['cpu_app','cpu_sys','mem_total','mem_native','mem_dalvik',
                             'battery_level', 'battery_tem','upflow','downflow','fps']
    ios_log_file_list = ['cpu_app','cpu_sys', 'mem_total', 'battery_tem', 'battery_current', 
                         'battery_voltage', 'battery_power','upflow','downflow','fps','gpu']
    log_file_list = android_log_file_list if platform == 'Android' else ios_log_file_list
    wb = xlwt.Workbook(encoding = 'utf-8')
   
    k = 1
    for name in log_file_list:
        ws1 = wb.add_sheet(name)
        ws1.write(0,0,'Time') 
        ws1.write(0,1,'Value')
        row = 1 #start row
        col = 0 #start col
        if os.path.exists(f'{self.report_dir}/{scene}/{name}.log'):
            f = open(f'{self.report_dir}/{scene}/{name}.log','r',encoding='utf-8')
            for lines in f: 
                target = lines.split('=')
                k += 1
                for i in range(len(target)):
                    ws1.write(row, col ,target[i])
                    col += 1
                row += 1
                col = 0
    xls_path = os.path.join(self.report_dir, scene, f'{scene}.xls')            
    wb.save(xls_path)
    logger.info('Exporting excel success : {}'.format(xls_path))
    return xls_path   
```



## 图片

通过html2canvas实现

```js
function screenshot(classname,downloadname) {
    let container = document.getElementsByClassName(classname); //page-wrapper
    html2canvas(container[0]).then(canvas => {
        let imgData = canvas.toDataURL();
        let link = document.createElement('a');
        link.href = imgData;
        link.download = downloadname;
        let flag = link.click();
        console.log(flag);
    })
}
```



## html

通过模板渲染成一个html文件，然后导出

```python
def make_android_html(self, scene, summary : dict):
    logger.info('Generating HTML ...')
    STATICPATH = os.path.dirname(os.path.realpath(__file__))
    file_loader = FileSystemLoader(os.path.join(STATICPATH, 'report_template'))
    env = Environment(loader=file_loader)
    template = env.get_template('android.html')
    with open(os.path.join(self.report_dir, scene, 'report.html'),'w+') as fout:
        html_content = template.render(cpu_app=summary['cpu_app'],cpu_sys=summary['cpu_sys'],
                                       mem_total=summary['mem_total'],mem_native=summary['mem_native'],
                                       mem_dalvik=summary['mem_dalvik'],fps=summary['fps'],
                                       jank=summary['jank'],level=summary['level'],
                                       tem=summary['tem'],net_send=summary['net_send'],
                                       net_recv=summary['net_recv'],cpu_charts=summary['cpu_charts'],
                                       mem_charts=summary['mem_charts'],net_charts=summary['net_charts'],
                                       battery_charts=summary['battery_charts'],fps_charts=summary['fps_charts'],
                                       jank_charts=summary['jank_charts'])
        
        fout.write(html_content)
    html_path = os.path.join(self.report_dir, scene, 'report.html')    
    logger.info('Generating HTML success : {}'.format(html_path))  
    return html_path
```



```python
def make_ios_html(self, scene, summary : dict):
    logger.info('Generating HTML ...')
    STATICPATH = os.path.dirname(os.path.realpath(__file__))
    file_loader = FileSystemLoader(os.path.join(STATICPATH, 'report_template'))
    env = Environment(loader=file_loader)
    template = env.get_template('ios.html')
    with open(os.path.join(self.report_dir, scene, 'report.html'),'w+') as fout:
        html_content = template.render(cpu_app=summary['cpu_app'],cpu_sys=summary['cpu_sys'],gpu=summary['gpu'],
                                       mem_total=summary['mem_total'],fps=summary['fps'],
                                       tem=summary['tem'],current=summary['current'],
                                       voltage=summary['voltage'],power=summary['power'],
                                       net_send=summary['net_send'],net_recv=summary['net_recv'],
                                       cpu_charts=summary['cpu_charts'],mem_charts=summary['mem_charts'],
                                       net_charts=summary['net_charts'],battery_charts=summary['battery_charts'],
                                       fps_charts=summary['fps_charts'],gpu_charts=summary['gpu_charts'])            
        fout.write(html_content)
    html_path = os.path.join(self.report_dir, scene, 'report.html')    
    logger.info('Generating HTML success : {}'.format(html_path))  
    return html_path
