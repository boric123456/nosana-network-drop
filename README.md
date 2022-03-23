# nosana-network-drop
/ jarvis.cpp
// Алгоритм Джарвиса ("заворачивания подарка")
// построения выпуклой оболочки множества точек на плоскости.
#include <cassert>
#include <utility> // swap
#include <cmath>   // abs
#include "ageo2d.hpp" // точки и вектора


/// В диапазоне точек найти самую верхнюю из самых правых.
Point_2* find_highest_rightmost(Point_2 *begin, Point_2 *end)
{
  assert(begin < end);
  double x_max = begin->x, y_max = begin->y;
  auto cur_max = begin;
  while (++begin != end)
  {
    if (x_max < begin->x
     || begin->x == x_max && y_max < begin->y)
    {
      x_max = begin->x;
      y_max = begin->y;
      cur_max = begin;
    }
  }

  return cur_max;
}


/// В диапазоне точек найти самый дальний поворот по часовой стрелке от точки v.
Point_2* max_cw_turn(Point_2 *begin, Point_2 *end, Point_2 v)
{
  assert(begin < end);
  auto cur_max = begin;
  auto vector = *begin - v; // воспользуемся оператором минус, определённым для точек выше
  while (++begin != end)
  {
    const auto new_vector = *begin - v;
    const auto cp = crossp(vector, new_vector);
    if (cp < 0.   // поворот от vector к new_vector по ЧС?
     || cp == 0.  // коллинеарны, но сонаправленны и new_vector длиннее, чем vector?
     && dotp(vector, vector) < dotp(vector, new_vector))
    {
      cur_max = begin;
      vector = new_vector;
    }
  }

  return cur_max;
}


/// Алгоритм заворачивания подарка.
/// Переставляет элементы исходного диапазона точек так,
/// чтобы вершины выпуклой оболочки в порядке обхода против часовой стрелки
/// встали в начале диапазона, возвращает указатель на следующую за последней
/// вершиной построенной выпуклой оболочки вершину.
Point_2* convex_hull_jarvis(Point_2 *begin, Point_2 *end)
{
  using std::swap;
  if (begin == end)
    return end;

  // Найти позицию самой верхней из самых правых точек.
  // Это -- последняя вершина выпуклой оболочки.
  auto cur = find_highest_rightmost(begin, end);
  // Потенциальная ошибка: если есть более одной точки, равной *cur,
  // то алгоритм может выдать некорректный результат.
  // Как можно исправить эту ситуацию?
  
  // Поставить её в конец последовательности для того,
  // чтобы корректно выйти из цикла, когда следующая вершина совпадёт с ней.
  const auto last_pos = end - 1;
  swap(*cur, *last_pos);
  cur = last_pos;

  // Цикл по вершинам выпуклой оболочки.
  while (true)
  {
    const auto next = max_cw_turn(begin, end, *cur);
    // Поставить следующую вершину.
    swap(*begin, *next);
    cur = begin++;

    if (next == last_pos) // Выпуклая оболочка построена.
      return begin;
  }
}


///////////////////////////////////////////////////////////////////////////////
// Геометрические операции, применяемые при тестировании

/// Проверка многоугольника на строгую выпуклость.
bool is_strictly_convex(const Point_2 *begin, const Point_2 *end)
{
  if (end - begin < 2)
    return true; // одна точка

  if (end - begin < 3)
    return begin[0] != begin[1]; // отрезок

  // Проходя по всем углам (парам смежных рёбер) многоугольника,
  // проверять, что поворот происходит строго против ЧС.
  auto a = *(end - 2), b = *(end - 1);
  do
  {
    const auto c = *begin++;
    const auto // рёбра
      ba = b - a,
      cb = c - b;

    // Проверить поворот от ba к cb.
    if (crossp(ba, cb) <= 0.)
      return false;

    a = b;
    b = c;
  } while (begin != end);

  return true;
}


/// Положение точки относительно множества.
enum Point_location
{
  point_location_inside,   // внутри
  point_location_boundary, // на границе
  point_location_outside   // снаружи
};

/// Определить положение точки p относительно многоугольника [begin, end),
/// в котором вершины перечислены в порядке обхода против часовой стрелки.
/// Осторожно: на результат может влиять погрешность вычислений.
/// Используется правило витков (== ненулевого индекса).
/// Алгоритм позаимствован с http://geomalgorithms.com/a03-_inclusion.html
Point_location locate_point
  (
    const Point_2 *begin, const Point_2 *end, // многоугольник
    Point_2 p,                                // точка
    double tolerance = 0.                     // условный допуск на границе
  )
{
  using std::abs;
  if (begin == end)
    return point_location_outside;

  int wn = 0; // количество витков
  // Проходя по всем рёбрам многоугольника, считать количество витков.
  auto prev = *(end - 1);
  do
  {
    const auto next = *begin++;
    const auto
      edge = next - prev,
      prad = p - prev;

    const auto cp = crossp(prad, edge);
    // Ребро пересекает луч снизу-вверх справа от точки p.
    if (prev.y <= p.y && p.y < next.y && 0. < cp)
      ++wn;
    // Ребро пересекает луч сверху-вниз справа от точки p.
    else if (next.y <= p.y && p.y < prev.y && cp < 0.)
      --wn;
    // Дополнительная проверка: точка лежит на ребре
    else if (abs(cp) <= tolerance
          && dotp(prad, prad) <= dotp(prad, edge))
      return point_location_boundary;

    prev = next;
  } while (begin != end);

  return wn == 0 ? point_location_outside : point_location_inside;
}


///////////////////////////////////////////////////////////////////////////////
// Тестирование
