return { code: (r[0] || '').trim(), marks: dates.map(function (_, j) {
        var v = (r[j + 1] || '').trim();
        return v === '1' ? 1 : v === '0' ? 0 : null;
      }) };
    }).filter(function (s) { return s.code !== ''; });

    if (!dates.length || !students.length) throw new Error('В файле нет занятий или студентов.');

    // Явка по каждому занятию: сумма / число заполненных ячеек
    var perLesson = dates.map(function (_, j) {
      var sum = 0, n = 0;
      students.forEach(function (s) { if (s.marks[j] !== null) { sum += s.marks[j]; n++; } });
      return n ? Math.round(sum / n * 1000) / 10 : null;
    });

    // Доля посещений каждого студента (только по занятиям, где он был в списке)
    var shares = [];
    students.forEach(function (s) {
      var sum = 0, n = 0;
      s.marks.forEach(function (m) { if (m !== null) { sum += m; n++; } });
      if (n) shares.push(sum / n);
    });

    // Средняя посещаемость по всем заполненным ячейкам
    var total = 0, cnt = 0;
    students.forEach(function (s) { s.marks.forEach(function (m) { if (m !== null) { total += m; cnt++; } }); });
    var avg = cnt ? Math.round(total / cnt * 1000) / 10 : 0;

    var buckets = [0, 0, 0, 0];
    shares.forEach(function (p) {
      if (p < 0.25) buckets[0]++;
      else if (p < 0.5) buckets[1]++;
      else if (p < 0.75) buckets[2]++;
      else buckets[3]++;
    });

    return { dates: dates, perLesson: perLesson, avg: avg, nStudents: students.length, buckets: buckets };
  }

  function render(d) {
    document.getElementById('k-avg').textContent = d.avg.toString().replace('.', ',') + ' %';
    document.getElementById('k-students').textContent = d.nStudents;
    document.getElementById('k-lessons').textContent = d.dates.length;

    var line = echarts.init(document.getElementById('line'));
    line.setOption({
      grid: { left: 44, right: 16, top: 24, bottom: 56 },
      tooltip: { trigger: 'axis', valueFormatter: function (v) { return v + ' %'; } },
      xAxis: { type: 'category', data: d.dates, axisLabel: { rotate: 45 } },
      yAxis: { type: 'value', min: 0, max: 100, axisLabel: { formatter: '{value} %' } },
      series: [
        { name: 'Явка', type: 'line', data: d.perLesson, smooth: false, symbolSize: 7,
          itemStyle: { color: '#0f766e' }, lineStyle: { width: 3 },
          markLine: { silent: true, symbol: 'none', lineStyle: { type: 'dashed', color: '#b45309', width: 2 },
            label: { formatter: 'среднее ' + d.avg + ' %', position: 'insideEndTop' },
            data: [{ yAxis: d.avg }] } }
      ]
    });

    var bars = echarts.init(document.getElementById('bars'));
    bars.setOption({
      grid: { left: 44, right: 16, top: 24, bottom: 40 },
      tooltip: { trigger: 'axis', axisPointer: { type: 'shadow' }, valueFormatter: function (v) { return v + ' студ.'; } },
      xAxis: { type: 'category', data: ['меньше 25 %', '25–50 %', '50–75 %', '75 % и больше'] },
      yAxis: { type: 'value', minInterval: 1, name: 'студентов' },
      series: [{ type: 'bar', data: d.buckets, barMaxWidth: 64,
        itemStyle: { color: '#0f766e' }, label: { show: true, position: 'top' } }]
    });

    window.addEventListener('resize', function () { line.resize(); bars.resize(); });
  }

  if (typeof echarts === 'undefined') {
    showError('Не загрузилась библиотека ECharts. Проверьте доступ к cdn.jsdelivr.net или положите echarts.min.js в репозиторий.');
    return;
  }

  fetch(CSV_FILE)
    .then(function (r) {
      if (!r.ok) throw new Error('Файл ' + CSV_FILE + ' не найден (код ' + r.status + ').');
      return r.arrayBuffer();
    })
    .then(function (buf) {
      var text = decode(buf);
      var rows = parseCSV(text, detectDelimiter(text));
      if (rows.length < 2) throw new Error('В файле меньше двух строк.');
      render(build(rows));
    })
    .catch(function (e) {
      showError('Не удалось показать данные: ' + e.message + ' Если страница открыта с диска двойным щелчком, откройте её по адресу github.io.');
    });
})();
</script>
</body>
</html>
