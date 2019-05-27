Parcel p = Parcel.obtain();
p.writeInt(1);
p.writeInt(2);

//读取前必须设置位置，否则写了两个int即8个字节，那么当前位置是9，则会从第9位置开始读，所以要先设为0
p.setDataPosition(0);

Log.e("测试测试", "onStart: " + p.readInt());
Log.e("测试测试", "onStart: " + p.readInt());
p.recycle();